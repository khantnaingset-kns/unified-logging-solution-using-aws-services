# Unified Logging Solution using AWS Services

Logging is the most important aspects of modern cloud and application developments. There are too many logging solutions out there, but if the services and applications are in AWS, it’s better to use this solution. Of course, there are several drawbacks, like cost, but by using this method, there is no need for extra setup; everything will take place inside one provider, which can easily manage security and scalability without too many overheads.

This article/repo is all about manual instructions and steps, and using IAC (Infrastructure as a Code) on how to construct a unified logging solution inside AWS using CloudWatch, Kinesis, Kinesis Firehose, and OpenSearch. Despite the fact that there are numerous logging solutions available, this one is best suited for organizations that rely heavily on AWS for infrastructure. There is no need to rely on other provider or services, everything can be done within the AWS.

![infra-diagram.png](/assets/images/infra-diagram.png)

Logs could come from anywhere, CloudTrail's, EC2, S3, Billing, or Custom Applications, but in this one, I’m about to use logs from EC2 Debian Based Instance, lambda functions, AWS Glue and CloudTrail.

## **Setup using AWS UI Wizard**

## Setup the Kinesis Data Stream

- Create the **Kinesis Data Stream** in **Amazon Kinesis.**
- And then,
- Create the **IAM Policy** with a name ****called **KinesisDataStreamAccessPolicy** for to access Kinesis Stream from **CloudWatch** with following permission

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

- Create the **IAM Role** with a name ****called **CloudWatchLogsToKinesisRole** with **Custom trust policy** role type by custom JSON **(Since the CloudWatch logs is not in already defined AWS Services, need to write custom policy)**

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

- During **IAM Role** creation, attached the **KinesisDataStreamAccessPolicy**,
- Name this IAM Policy as

## AWS Glue Logs from CloudWatch to Amazon Kinesis

- Create the IAM Role with **AWSGlueServiceRole**
- Go to **AWS Glue Studio** Console
- Click on **Jobs**
- Create the **Job** with **Visual with a source and target** template
- Go to **Visual** tab
- Click on **Data source - S3 bucket** block and Choose **data source** from **S3 Bucket** (Need to upload some dataset in CSV format into S3!!!)
- Click on **Apply Mapping** Block and configure the desire output of dataset in **Apply Mapping. (**Can view sample output in this menu.)
- Click on **Data target - S3 bucket**  and configure for the processed data target,
- Go to **Job details** tab
- Attached the created **IAM Role**

After that save the job and click on **Run**. Every logs generated by glue will be in **CloudWatch.** To know specific location click on created job under **Jobs** tab and go to **Runs.** Under the run there is the link for **CloudWatch** logs location.

- After running the jobs go to glue log groups in **CloudWatch** and select on log group and click on **Action**
- Select to **Action Subscription Filters.**
- To configure the Destination choose **Create Kinesis Subscription Filter**.
    - Select **Current Account**.
    - Select your Kinesis data stream from the dropdown list.
    - In the IAM Role selection select **CloudWatchLogsToKinesisRole.**
- Review the **Distribution method**:
    - **By Log Stream** - Verifies that downstream consumers can 
    aggregate log events by log stream, but might be less efficient. This 
    method might also incur higher streaming costs because it requires more 
    shards.
    - **Random** - Distributes the load across Kinesis stream shards, but downstream consumers can't aggregate log events by log stream.
- Configure log format and named the filter (Filter is the main point in streaming to other services)
- Verify the pattern
- Click **Start Streaming**

## AWS CloudTrail Logs From CloudWatch to Amazon Kinesis

- Go to **CloudTrail** Dashboard
- Click on **Create trail**
- Give **Trail name** as **GeneralTrail**
- Select **Create new S3 bucket** and give new bucket name
- Check **Enabled** on **Log file SSE-KMS encryption** for better security
- Choose **New** on **Customer managed AWS KMS key** and give name for new KMS key
- Check **Enabled** on **Log file validation**
- Check **Enabled** on **CloudWatch Logs**
    - Give CloudWatch Logs group name as **GeneralTrailLogGroup**
    - Give new IAM Role name as **GeneralTrailCloudWatchRole**
- Click **Next**
- Check on all **Event type** so CloudTrail can collect logs for everything. On **Data events** and **Insights events** choose prefer services. (On Data events **CloudTrail** can collect everything from **Lambda, DynamoDB** and **RDS** as well)
- Click **Next** and Create the **CloudTrail**

After above steps

- Go to the **CloudWatch** and select the **GeneralTrailLogGroup**
- Select to **Action Subscription Filters.**
- To configure the Destination choose **Create Kinesis Subscription Filter**.
    - Select **Current Account**.
    - Select your Kinesis data stream from the dropdown list.
    - In the IAM Role selection select **CloudWatchLogsToKinesisRole.**
- Review the **Distribution method**:
    - **By Log Stream** - Verifies that downstream consumers can 
    aggregate log events by log stream, but might be less efficient. This 
    method might also incur higher streaming costs because it requires more 
    shards.
    - **Random** - Distributes the load across Kinesis stream shards, but downstream consumers can't aggregate log events by log stream.
- Configure log format and named the filter (Filter is the main point in streaming to other services)
- Verify the pattern
- Click **Start Streaming**

## **Ec2 Logs from CloudWatch to Amazon Kinesis**

- Create **EC2 Instance**
- Create the **IAM Role**  with the name called **EC2CloudWatchAccessRole CloudWatchAgentServerPolicy.**
- Attached the created **EC2CloudWatchAccessRole** to EC2 Instance
- Install **CloudWatch agent**

   

**For Ubuntu or Debian**

```bash
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
```

```bash
sudo dpkg -i -E ./amazon-cloudwatch-agent.deb
```

For Amazon Linux or RPM

```bash
sudo yum install amazon-cloudwatch-agent -y
```

- Run the cloud watch agent wizard, (This will ask series of questions, and will configure CloudWatch agent by itself)

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

- Point to the system logs file in Wizard: `/var/log/syslog` in Ubuntu or Debian and `/var/log/boot.log` in amazon Linux or RPM based
- Start the agent

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json -s
```

- Now, log can be view in CloudWatch Logs Group by configure logs group name.
- Go to the **CloudWatch** and select the EC2 **Log Group**
- Select to **Action Subscription Filters.**
- To configure the Destination choose **Create Kinesis Subscription Filter**.
    - Select **Current Account**.
    - Select your Kinesis data stream from the dropdown list.
    - In the IAM Role selection select **CloudWatchLogsToKinesisRole.**
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
    - Function is rather simple but had to add UTF8 to base 64 conversion, otherwise **firehose** will show delivery error

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
    

When all of the above steps are finished, need login into OpenSearch using master username and password. Inside the OpenSearch need to do few configuration. Since, F**ine grant control** is enable at OpenSearch, need to grant permissions to **Firehose Delivery** to deliver the data to **OpenSearch.**

- Login into **OpenSearch** Dashboard using **master username and password**
- Go the **Security** and Clicks **Role**
- In the role section, click on **Create role**
- Give the role name as **firehose-role**
- For cluster permissions, add `cluster_composite_ops` for cluster operation
 and `cluster_monitor` for monitoring the cluster
- Under **Index permissions** choose **Index Patterns** and enter `unified-logs*`
- Under **Permissions,** add three action groups: `crud` `create_index`and `manage`
- Click **Create Role**

And then

- Under **Security**, choose **Role Mappings**
- Choose the role you just created (`firehose-role`).
- For **Backend Roles**, choose **Add Backend Role**.
- Enter the IAM ARN of the role Kinesis Data Firehose uses: `arn:aws:iam::<accountid>:role/<*firehose_stream_role_name>*`.