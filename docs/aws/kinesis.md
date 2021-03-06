# AWS Kinesis

## What is streaming data?

Streaming data is data that is generated continuously by thousands of data sources,
which typically send the data records simultaneously, and in small sizes (order of Kilobytes)/

* Purchases from online stores (amazong.com)
* Stock Prices
* Game data (as the gamer plays)
* Social network data
* Geospatial data (uber.com)
* iOT sensor data

## What is Kinesis?

Amazon Kinesis is a platform on AWS to send your streaming data too.
Kinesis makes it easy to load and analyze streaming data, and also providing the ability
for you to build your own custom applications for your business needs.

## What are the core Kinesis Services

* Kinesis Streams (video & data)
* Kinesis Firehose
* Kinesis Analytics

## Kinesis Streams

* Producers -> Kinesis Streams
* Stores data in shards.
* Data stored 24hrs, can be upped to 7 days
* Consumers -> EC2 instances taking data from shards & doing analysis
* Data outputted to DynamoDB, S3, EMR, Redshift etc.

* Kinesis Streams consist of shards
    * 5 transactions per second for reads, up to max total data read rate of 2MB per second
    * up to 1,000 records per second for writes, up to a maximum total data write rate of 1MB per second (including partition keys)
    * The data capacity of your stream is a function of the number of shards that you specify for the stream.
    The total capacity of the stream is the sum of the capacities of its shards.

## Kinesis Firehouse

* Producers -> Kinesis Firehose (no concern for shards/streams as its automated)
* No data retention period -> once data enters its optionally analyzed via lambda or outputted (i.e. ElasticSearch Cluster / S3 -> RedShift)
* No concern for consumers (lambda analytics if you wish - optional) -> output S3

## Kinesis Analytics

* Allows you to run SQL queries of data as it exists within Firehouse/Streams to output data inside Kinesis/Streams to S3/Redshift/ElasticSearch

## Exam Tips

* Know the difference between Kinesis Streams and Firehose. You will be given scenario questions and you must choose the most relevant service
* Understand what Kinesis Analytics is

## Sample CloudFormation Deployment with Kinesis

```javascript
{
  "AWSTemplateFormatVersion" : "2010-09-09",

  "Description" : "The Amazon Kinesis Data Visualization Sample Application",

  "Parameters" : {
    "InstanceType" : {
      "Description" : "EC2 instance type",
      "Type" : "String",
      "Default" : "t2.micro",
      "AllowedValues" : [ "t2.micro", "t2.small", "t2.medium", "m3.medium", "m3.large", "m3.xlarge", "m3.2xlarge", "c3.large", "c3.xlarge", "c3.2xlarge", "c3.4xlarge", "c3.8xlarge" ],
      "ConstraintDescription" : "must be a supported EC2 instance type for this template."
    },

    "KeyName" : {
      "Description" : "(Optional) Name of an existing EC2 KeyPair to enable SSH access to the instance. If this is not provided you will not be able to SSH on to the EC2 instance.",
      "Type" : "String",
      "Default" : "",
      "MinLength" : "0",
      "MaxLength" : "255",
      "AllowedPattern" : "[\\x20-\\x7E]*",
      "ConstraintDescription" : "can contain only ASCII characters."
    },

    "SSHLocation" : {
      "Description" : "The IP address range that can be used to SSH to the EC2 instances",
      "Type" : "String",
      "MinLength" : "9",
      "MaxLength" : "18",
      "Default" : "0.0.0.0/0",
      "AllowedPattern" : "(\\d{1,3})\\.(\\d{1,3})\\.(\\d{1,3})\\.(\\d{1,3})/(\\d{1,2})",
      "ConstraintDescription" : "must be a valid IP CIDR range of the form x.x.x.x/x."
    },

    "ApplicationArchive" : {
      "Description" : "A publicly accessible URL to the sample application archive as produced by 'mvn package'",
      "Type" : "String",
      "MinLength" : "7",
      "MaxLength" : "255",
      "Default" : "https://github.com/awslabs/amazon-kinesis-data-visualization-sample/releases/download/v1.1.1/amazon-kinesis-data-visualization-sample-1.1.1-assembly.zip"
    }
  },

  "Conditions": {
    "UseEC2KeyName": {"Fn::Not": [{"Fn::Equals" : [{"Ref" : "KeyName"}, ""]}]}
  },

  "Mappings" : {
    "AWSInstanceType2Arch" : {
      "t2.micro"    : { "Arch" : "64" },
      "t2.small"    : { "Arch" : "64" },
      "t2.medium"   : { "Arch" : "64" },
      "m3.medium"   : { "Arch" : "64" },
      "m3.large"    : { "Arch" : "64" },
      "m3.xlarge"   : { "Arch" : "64" },
      "m3.2xlarge"  : { "Arch" : "64" },
      "c3.large"    : { "Arch" : "64" },
      "c3.xlarge"   : { "Arch" : "64" },
      "c3.2xlarge"  : { "Arch" : "64" },
      "c3.4xlarge"  : { "Arch" : "64" },
      "c3.8xlarge"  : { "Arch" : "64" }
    },

    "AWSRegionArch2AMI" : {
      "us-east-1"      : { "64" : "ami-76817c1e" },
      "us-west-2"      : { "64" : "ami-d13845e1" },
      "eu-west-1"      : { "64" : "ami-892fe1fe" },
      "ap-southeast-1" : { "64" : "ami-a6b6eaf4" },
      "ap-southeast-2" : { "64" : "ami-d9fe9be3" },
      "ap-northeast-1" : { "64" : "ami-29dc9228" }
    }
  },

  "Resources" : {
    "KinesisStream" : {
      "Type" : "AWS::Kinesis::Stream",
      "Properties" : {
        "ShardCount" : "2"
      }
    },

    "KCLDynamoDBTable" : {
      "Type" : "AWS::DynamoDB::Table",
      "Properties" : {
        "AttributeDefinitions" : [
          {
            "AttributeName" : "leaseKey",
            "AttributeType" : "S"
          }
        ],
        "KeySchema" : [
          {
            "AttributeName" : "leaseKey",
            "KeyType" : "HASH"
          }
        ],
        "ProvisionedThroughput" : {
          "ReadCapacityUnits" : "10",
          "WriteCapacityUnits" : "5"
        }
      }
    },

    "CountsDynamoDBTable" : {
      "Type" : "AWS::DynamoDB::Table",
      "Properties" : {
        "AttributeDefinitions" : [
          {
            "AttributeName" : "resource",
            "AttributeType" : "S"
          },
          {
            "AttributeName" : "timestamp",
            "AttributeType" : "S"
          }
        ],
        "KeySchema" : [
          {
            "AttributeName" : "resource",
            "KeyType" : "HASH"
          },
          {
            "AttributeName" : "timestamp",
            "KeyType" : "RANGE"
          }
        ],
        "ProvisionedThroughput" : {
          "ReadCapacityUnits" : "10",
          "WriteCapacityUnits" : "5"
        }
      }
    },

    "Ec2SecurityGroup" : {
      "Type" : "AWS::EC2::SecurityGroup",
      "Properties" : {
        "GroupDescription" : "Enable SSH access and HTTP access on the inbound port",
        "SecurityGroupIngress" :
          [{ "IpProtocol" : "tcp", "FromPort" : "22", "ToPort" : "22", "CidrIp" : { "Ref" : "SSHLocation"} },
           { "IpProtocol" : "tcp", "FromPort" : "80", "ToPort" : "80", "CidrIp" : "0.0.0.0/0"}]
      }
    },

    "EIP" : {
      "Type" : "AWS::EC2::EIP",
      "Properties" : {
        "InstanceId" : { "Ref" : "Ec2Instance" }
      }
    },

    "RootRole": {
       "Type" : "AWS::IAM::Role",
       "Properties" : {
          "AssumeRolePolicyDocument": {
             "Version" : "2012-10-17",
             "Statement" : [ {
                "Effect" : "Allow",
                "Principal" : {
                   "Service" : [ "ec2.amazonaws.com" ]
                },
                "Action" : [ "sts:AssumeRole" ]
             } ]
          },
          "Path" : "/"
       }
    },

    "RolePolicies" : {
       "Type" : "AWS::IAM::Policy",
       "Properties" : {
          "PolicyName" : "root",
          "PolicyDocument" : {
             "Version" : "2012-10-17",
             "Statement" : [ {
                "Effect" : "Allow",
                "Action" : "kinesis:*",
                "Resource" : { "Fn::Join" : [ "", [ "arn:aws:kinesis:", { "Ref" : "AWS::Region" }, ":", { "Ref" : "AWS::AccountId" }, ":stream/", { "Ref" : "KinesisStream" } ]]}
             }, {
                "Effect" : "Allow",
                "Action" : "dynamodb:*",
                "Resource" : { "Fn::Join" : [ "", [ "arn:aws:dynamodb:", { "Ref" : "AWS::Region" }, ":", { "Ref" : "AWS::AccountId" }, ":table/", { "Ref" : "KCLDynamoDBTable" } ]]}
             }, {
                "Effect" : "Allow",
                "Action" : "dynamodb:*",
                "Resource" : { "Fn::Join" : [ "", [ "arn:aws:dynamodb:", { "Ref" : "AWS::Region" }, ":", { "Ref" : "AWS::AccountId" }, ":table/", { "Ref" : "CountsDynamoDBTable" } ]]}
             }, {
                "Effect" : "Allow",
                "Action" : "cloudwatch:*",
                "Resource" : "*"
             } ]
          },
          "Roles" : [ { "Ref": "RootRole" } ]
       }
    },

    "RootInstanceProfile" : {
       "Type" : "AWS::IAM::InstanceProfile",
       "Properties" : {
          "Path" : "/",
          "Roles" : [ { "Ref": "RootRole" } ]
       }
    },

    "Ec2Instance": {
      "Type" : "AWS::EC2::Instance",
      "Metadata" : {
        "AWS::CloudFormation::Init" : {
          "config" : {
            "packages" : {
              "yum" : {
                "java-1.7.0-openjdk" : []
              }
            },
            "files" : {
              "/var/kinesis-data-vis-sample-app/watchdog.sh" : {
                "content" : {"Fn::Join" : ["", [
                  "#!/bin/bash\n",
                  "if ! ps aux | grep HttpReferrerCounterApplication | grep -v grep ; then\n",
                  "    # Launch the Kinesis application for counting HTTP referrer pairs\n",
                  "    java -cp /var/kinesis-data-vis-sample-app/lib/\\* com.amazonaws.services.kinesis.samples.datavis.HttpReferrerCounterApplication ", { "Ref" : "KCLDynamoDBTable" }, " ", { "Ref" : "KinesisStream" }, " ", { "Ref" : "CountsDynamoDBTable" }, " ", { "Ref" : "AWS::Region" }, " &>> /home/ec2-user/kinesis-data-vis-sample-app-kcl.log &\n",
                  "fi\n",
                  "if ! ps aux | grep HttpReferrerStreamWriter | grep -v grep ; then\n",
                  "    # Launch our Kinesis stream writer to fill our stream with generated HTTP (resource, referrer) pairs.\n",
                  "    # This will create a writer with 5 threads to send records indefinitely.\n",
                  "    java -cp /var/kinesis-data-vis-sample-app/lib/\\* com.amazonaws.services.kinesis.samples.datavis.HttpReferrerStreamWriter 5 ",  { "Ref" : "KinesisStream" }, " ", { "Ref" : "AWS::Region" }, " &>> /home/ec2-user/kinesis-data-vis-sample-app-publisher.log &\n",
                  "fi\n",
                  "if ! ps aux | grep WebServer | grep -v grep ; then\n",
                  "    # Launch the webserver\n",
                  "    java -cp /var/kinesis-data-vis-sample-app/lib/\\* com.amazonaws.services.kinesis.samples.datavis.WebServer 80 /var/kinesis-data-vis-sample-app/wwwroot ", { "Ref" : "CountsDynamoDBTable" }, " ", { "Ref" : "AWS::Region" }, " &>> /home/ec2-user/kinesis-data-vis-sample-app-www.log &\n",
                  "fi\n"
                ]]},
                "mode" : "000755",
                "owner" : "ec2-user",
                "group" : "ec2-user"
              },
              "/var/kinesis-data-vis-sample-app/crontask" : {
                "content" : {"Fn::Join" : ["", [
                  "* * * * * bash /var/kinesis-data-vis-sample-app/watchdog.sh\n"
                ]]},
                "mode" : "000644",
                "owner" : "ec2-user",
                "group" : "ec2-user"
              }
            },
            "sources": {
              "/var/kinesis-data-vis-sample-app" : { "Ref" : "ApplicationArchive" }
            }
          }
        }
      },

      "Properties" : {
        "KeyName" : { "Fn::If" : [ "UseEC2KeyName", { "Ref" : "KeyName" }, { "Ref" : "AWS::NoValue" } ]},
        "ImageId" : { "Fn::FindInMap" : [ "AWSRegionArch2AMI", { "Ref" : "AWS::Region" },
                                          { "Fn::FindInMap" : [ "AWSInstanceType2Arch", { "Ref" : "InstanceType" },
                                          "Arch" ] } ] },
        "InstanceType" : { "Ref" : "InstanceType" },
        "SecurityGroups" : [{ "Ref" : "Ec2SecurityGroup" }],
        "IamInstanceProfile": { "Ref": "RootInstanceProfile" },
        "UserData"       : { "Fn::Base64" : { "Fn::Join" : ["", [
          "#!/bin/bash\n",
          "yum update -y aws-cfn-bootstrap\n",

          "/opt/aws/bin/cfn-init -s ", { "Ref" : "AWS::StackId" }, " -r Ec2Instance ",
          "         --region ", { "Ref" : "AWS::Region" }, "\n",

          "# Register watchdog script with cron\n",
          "crontab /var/kinesis-data-vis-sample-app/crontask\n",

          "# Launch watchdog script immediately so if it fails this stack fails to start\n",
          "/var/kinesis-data-vis-sample-app/watchdog.sh\n",

          "/opt/aws/bin/cfn-signal -e $? '", { "Ref" : "WaitHandle" }, "'\n"
        ]]}}
      }
    },

    "WaitHandle" : {
      "Type" : "AWS::CloudFormation::WaitConditionHandle"
    },

    "WaitCondition" : {
      "Type" : "AWS::CloudFormation::WaitCondition",
      "DependsOn" : "Ec2Instance",
      "Properties" : {
        "Handle" : {"Ref" : "WaitHandle"},
        "Timeout" : "600"
      }
    }
  },
  "Outputs" : {
    "URL" : {
      "Description" : "URL to the sample application's visualization",
      "Value" : { "Fn::Join" : [ "", [ "http://", { "Fn::GetAtt" : [ "Ec2Instance", "PublicDnsName" ] }]]}
    },
    "InstanceId" : {
      "Description" : "InstanceId of the newly created EC2 instance",
      "Value" : { "Ref" : "Ec2Instance" }
    },
    "AZ" : {
      "Description" : "Availability Zone of the newly created EC2 instance",
      "Value" : { "Fn::GetAtt" : [ "Ec2Instance", "AvailabilityZone" ] }
    },
    "StreamName" : {
      "Description" : "The name of the Kinesis Stream. This was autogenerated by the Kinesis Resource named 'KinesisStream'",
      "Value" : { "Ref" : "KinesisStream" }
    },
    "ApplicationName" : {
      "Description" : "The name of the Kinesis Client Application. This was autogenerated by the DynamoDB Resource named 'KCLDynamoDBTable'",
      "Value" : { "Ref" : "KCLDynamoDBTable" }
    },
    "CountsTable" : {
      "Description" : "The name of the DynamoDB table where counts are persisted. This was autogenerated by the DynamoDB Resource named 'CountsDynamoDBTable'",
      "Value" : { "Ref" : "CountsDynamoDBTable" }
    }
  }
}
```

## Kinesis Shards -v- Consumers

* Kinesis Client Library runs on the consumer instances
* Tracks the number of shards in your stream
* Discovers new shards when you reshard

## Kinesis Client Library

* The KCL ensures that for every shard there is a record processor
* Manages the number of record processors relative to the number of shards & consumers
* If you have only one consumer, the KCL will create all the record processors on a single consumer
* If you have two consumers it will load balance and create half the processors on one instance
and half on another

## Scaling out the Consumers

* With KCL, generally you should ensure that the number of instances does not exceed
the number of shards (except for failure or standby purposes)
* You never need multiple instances to handle the processing of one shard
* However, one worker can process multiple shards
* It's fine if the number of shards exceeds the number of instances
* Don't think that just because you reshard, that you need to add more instances
* Instead, CPU utilization is what should drive the quantity of consumer instances
you have, NOT the number of shards in your Kinesis stream
* Use an Auto Scaling group, and base scaling decisions on CPU load on your consumers

## Kinesis Shards - Exam Tips

* The Kinesis Client Library running on your consumers creates a record processor for each
shard that is being consumed by your instances
* If you increase the number of shards, the KCL will add more record processors on your consumers
* CPU utilization is what should drive the quantity of consumer instances you have, NOT the
number of shards in your Kinesis stream
* Use an autoscaling group, and base scaling decisions on CPU load on your consumers
