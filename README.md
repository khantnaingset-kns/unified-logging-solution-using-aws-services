# Unified Logging Solution using AWS Services

This article/repo is all about instructions, steps, and usage of IAC (Infrastructure as a Code) on how to construct a unified logging solution inside AWS using CloudWatch, Kinesis, Kinesis Firehose, and OpenSearch. Despite the fact that there are numerous logging solutions available, this one is best suited for organizations that rely heavily on AWS for infrastructure. There is no need to rely on other provider or services, everything can be done within the AWS.

![infra-diagram.png](/assets/images/infra-diagram.png)

Logs could come from anywhere, CloudTrail's, Ec2, S3, Billing, or Custom Applications, but in this one, I’m about to use logs from Ec2 Debian Based Instance as example.

## **Setup using AWS UI Wizard**

**Ec2 Logs to CloudWatch**

- Create **Ec2 Instance**
- Create the **IAM Role** with **CloudWatchAgentServerPolicy.**
- Attached the created **IAM Role** to Ec2 Instance
- Install **CloudWatch agent**

   

```bash
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
```

```bash
sudo dpkg -i -E ./amazon-cloudwatch-agent.deb
```

- Run the cloud watch agent wizard, (This will ask series of questions, and will configure CloudWatch agent by itself)

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

- Start the agent

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json -s
```

- Now, log can be view in CloudWatch Logs Group by configure logs group name.

**CloudWatch to Amazon Kinesis**

- Create the **Kinesis Data Stream** in **Amazon Kinesis.**
- Create the **IAM Policy** for to access Kinesis Stream from **CloudWatch** with following permission

```json
{
	"Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "kinesis:PutRecord",
    "Resource": "arn:aws:kinesis:<REGION>:<ACCOUNT_ID>:stream/<STREAM_NAME>"
  }]
}
```

- Create the **IAM Role** with **Custom trust policy** role type by custom JSON **(Since the CloudWatch logs is not in already defined AWS Services, need to write custom policy)**

```json
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Principal": {
      "Service": "logs.<REGION>.amazonaws.com"
    },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringLike": {
        "aws:SourceArn": "arn:aws:logs:<REGION>:<ACCOUNT_ID>:*"
      }
    }
  }
}
```

- During **IAM Role** creation, attached the above **IAM Policy**,
- Go to the **CloudWatch** and select the desired **Log Group**
- Select to **Action Subscription Filters.**
- To configure the Destination choose **Create Kinesis Subscription Filter**.
    - Select **Current Account**.
    - Select your Kinesis data stream from the dropdown list.
    - Select the IAM role that you previously created.
- Review the **Distribution method**:
    - **By Log Stream** - Verifies that downstream consumers can 
    aggregate log events by log stream, but might be less efficient. This 
    method might also incur higher streaming costs because it requires more 
    shards.
    - **Random** - Distributes the load across Kinesis stream shards, but downstream consumers can't aggregate log events by log stream.
- Configure log format and named the filter (Filter is the main point in streaming to other services)
- Verify the pattern
- Click **Start Streaming**

**Creating S3 for Data Stored and AWS Lambda for Data Transformation**

- Go to AWS **S3**
    - Create **bucket** for storing the **firehose logs backup**
- Go to **AWS Lambda**
    - Create function to process **Kinesis Logs** aka ETL Function**; using the following JavaScript code** below. **(Following code only work in Node.js 18 runtime or above )**

```jsx
import zlib from "zlib";

export const handler = async (event, context) => {
  let success = 0; // Number of valid entries found
  let failure = 0; // Number of invalid entries found
  /* Process the list of records */
  const output = event.records.map((record) => {
    try {
      const result = zlib
        .gunzipSync(Buffer.from(record.data, "base64"))
        .toString();
      console.log(result);
      success++;
      return {
        recordId: record.recordId,
        result: "Ok",
        data: new Buffer(result).toString("base64"),
      };
    } catch (e) {
      console.log(e);
      failure++;
    }
  });
  console.log(
    `Processing completed.  Successful records ${success}, Failed records ${failure}.`
  );
  return { records: output };
};
```

      

**Configure the Firehose and OpenSearch**

For the better security OpenSearch will run inside VPC, Kinesis Data Firehose can now deliver data into an Amazon OpenSearch Service VPC endpoint. This provides a secure and easy way to ingest, transform, and deliver streaming data. You don’t need to worry about managing your data ingestion and delivery infrastructure. With this new feature, Kinesis Data Firehose enables additional secure communication to Amazon OpenSearch Service VPC endpoints. Amazon OpenSearch Service endpoints that live within a VPC give you an extra layer of security.

When you create a Kinesis Data Firehose delivery stream that delivers data to an Amazon OpenSearch Service VPC endpoint, Kinesis Data Firehose creates an Elastic Network Interface (ENI) in each subnet you select. If you only use one Availability Zone, Kinesis Data Firehose places an endpoint into only one subnet. Similarly, when you create an Amazon OpenSearch Service VPC endpoint, it creates endpoints in the subnets you chose. Kinesis Data Firehose uses ENI to deliver the data to your Amazon OpenSearch Service ENI, all inside your VPC. 

In order to ingest data into OpenSearch which is inside the VPC, need to create multiple security group inside the VPC. At first just create two empty security groups, The one is for Firehose to deliver the stream data and the one for OpenSearch to use. 

**Kinesis Firehose Delivery Security Group**

| Inbound (empty) |  |  |  |
| --- | --- | --- | --- |
| Outbound |  |  |  |
| Type | Protocol | Port Range | Destination |
| HTTPS | TCP | 443 |  |

**OpenSearch Security Group**

| Inbound |  |  |  |
| --- | --- | --- | --- |
| Type | Protocol | Port | Destination |
| HTTP | TCP | 80 | <Bastion Host IP> |
| HTTPS | TCP | 443 | <Bastion Host IP> |
| Outbound |  |  |  |
| Type | Protocol | Port Range | Destination |
| All Traffic | All | All |  |

After creating two security groups, need to link them using HTTPS rules like below

**Kinesis Firehose Delivery Security Group (Linked)**

| Inbound (empty) |  |  |  |
| --- | --- | --- | --- |
| Outbound |  |  |  |
| Type | Protocol | Port Range | Destination |
| HTTPS | TCP | 443 | <OpenSearch Security Group> |

**OpenSearch Security Group (Linked)**

| Inbound |  |  |  |
| --- | --- | --- | --- |
| Type | Protocol | Port | Destination |
| HTTP | TCP | 80 | <Bastion Host IP> |
| HTTPS | TCP | 443 | <Bastion Host IP> |
| HTTPS | TCP | 443 | <Firehose Security Group> |
| Outbound |  |  |  |
| Type | Protocol | Port Range | Destination |
| All Traffic | All | All |  |

After creating the security groups:

- Go to **Amazon OpenSearch Service**
    - Click on **Domains** and then click on **Create Domain**
    - Give desire **domain name**
    - Enable **Custom endpoint** or not, (Disable for POC, too complicated for now)
    - Choose **Deployment Type**
    - Choose **Version** (Only Elasticsearch support dashboard right now)
    - Choose **Auto Tune**
    - Choose **Data Nodes**
    - 
    - In **Network** section, choose VPC access
        - Select **VPC**
        - Select **Subnets**
        - Select created **OpenSearch Security Group** from above
    - Choose **Fine-grained access control**
    - Choose **Create master user** and then create master user
    - Create the **OpenSearch Domain**

- Go the **Amazon Kinesis Firehose**
- Click on **Create Delivery Stream**
    - Choose **Source** and **Destination (**Source: Amazon Kinesis Data Streams and Destination: Amazon OpenSearch Service for current setup**)**
    - Choose **Kinesis data stream**
    - Type **Delivery stream name**
    - Enable **Transform records**, choose created **Lambda Function**
    - Choose the created **OpenSearch Domain**
    - Named the index as **unified-logs** and in index rotation choose **daily rotation**
    - Configure VPC as in same VPC as the OpenSearch
    - Choose **Subnet**
    - Select created **Kinesis Firehose Security Group** from above
    - In **Backup Settings** choose created **S3**
    - Click on **Create delivery stream**
    

When all of the above steps are finished, need login into OpenSearch using master username and password. Inside the OpenSearch need to do few configuration. Since, **Fine grant control** is enable at OpenSearch, need to grant permissions to **Firehose Delivery** to deliver the data to **OpenSearch.**

- Login into **OpenSearch** Dashboard using **master username and password**
- Go the **Security** and Clicks **Role**
- In the role section, click on **Create role**
- Give the role name as **firehose-role**
- For cluster permissions, add `cluster_composite_ops`
 and `cluster_monitor`
- Under **Index permissions** choose **Index Patterns** and enter `unified-logs*`
- Under **Permissions,** add three action groups: `crud` `create_index`and `manage`
- Choose **Save Role Definition**

And then

- Under **Security**, choose **Role Mappings**
- Choose the role you just created (`firehose-role`).
- For **Backend Roles**, choose **Add Backend Role**.
- Enter the IAM ARN of the role Kinesis Data Firehose uses: `arn:aws:iam::<accountid>:role/<*firehose_stream_role_name>*`.