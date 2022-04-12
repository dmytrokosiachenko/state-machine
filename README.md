# Workflow Studio compatible State Machine

This is a Workflow Studio compatible AWS Step Function state machine construct.

The goal of this construct is to make it easy to build and maintain your state machines using the Workflow Studio but still
leverage the AWS CDK as the source of truth for the state machine.

Read more about it [here](https://matthewbonig.com/2022/02/19/step-functions-and-the-cdk/).

## How to Use This Construct

Start by designing your initial state machine using the Workflow Studio.
When done with your first draft, copy and paste the ASL definition to a local file.

Create a new instance of this construct, handing it a fully parsed version of the ASL. 
Then add overridden values. 
The fields in the `overrides` field should match the `States` field of the ASL.

### Examples

```ts
const secret = new Secret(stack, 'Secret', {});
new StateMachine(stack, 'Test', {
  stateMachineName: 'A nice state machine',
  definition: JSON.parse(fs.readFileSync(path.join(__dirname, 'sample.json'), 'utf8').toString()),
  overrides: {
    'Read database credentials secret': {
      Parameters: {
        SecretId: secret.secretArn,
      },
    },
  },
});
```

You can also override nested states in arrays, for example:

```ts
new StateMachine(stack, 'Test', {
    stateMachineName: 'A-nice-state-machine',
    overrides: {
      Branches: [{
        StartAt: 'ResumeCluster',
        States: {
          ResumeCluster: {
            Parameters: {
              ClusterIdentifier: 'CLUSTER_NAME',
            },
          },
          DescribeClusters: {
            Parameters: {
              ClusterIdentifier: 'CLUSTER_NAME',
            },
          },
        },
      }, {
        StartAt: 'StartInstances',
        States: {
          StartInstances: {
            Parameters: {
              InstanceIds: ['INSTANCE_ID'],
            },
          },
          DescribeInstanceStatus: {
            Parameters: {
              InstanceIds: ['INSTANCE_ID'],
            },
          },
        },
      }],
    },
    stateMachineType: StateMachineType.STANDARD,
    definition: {
      States: {
        Branches: [
          {
            StartAt: 'ResumeCluster',
            States: {
              'ResumeCluster': {
                Type: 'Task',
                Parameters: {
                  ClusterIdentifier: 'MyData',
                },
                Resource: 'arn:aws:states:::aws-sdk:redshift:resumeCluster',
                Next: 'DescribeClusters',
              },
              'DescribeClusters': {
                Type: 'Task',
                Parameters: {
                  ClusterIdentifier: '',
                },
                Resource: 'arn:aws:states:::aws-sdk:redshift:describeClusters',
                Next: 'Evaluate Cluster Status',
              },
              'Evaluate Cluster Status': {
                Type: 'Choice',
                Choices: [
                  {
                    Variable: '$.Clusters[0].ClusterStatus',
                    StringEquals: 'available',
                    Next: 'Redshift Pass',
                  },
                ],
                Default: 'Redshift Wait',
              },
              'Redshift Pass': {
                Type: 'Pass',
                End: true,
              },
              'Redshift Wait': {
                Type: 'Wait',
                Seconds: 5,
                Next: 'DescribeClusters',
              },
            },
          },
          {
            StartAt: 'StartInstances',
            States: {
              'StartInstances': {
                Type: 'Task',
                Parameters: {
                  InstanceIds: [
                    'MyData',
                  ],
                },
                Resource: 'arn:aws:states:::aws-sdk:ec2:startInstances',
                Next: 'DescribeInstanceStatus',
              },
              'DescribeInstanceStatus': {
                Type: 'Task',
                Next: 'Evaluate Instance Status',
                Parameters: {
                  InstanceIds: [
                    'MyData',
                  ],
                },
                Resource: 'arn:aws:states:::aws-sdk:ec2:describeInstanceStatus',
              },
              'Evaluate Instance Status': {
                Type: 'Choice',
                Choices: [
                  {
                    And: [
                      {
                        Variable: '$.InstanceStatuses[0].InstanceState.Name',
                        StringEquals: 'running',
                      },
                      {
                        Variable: '$.InstanceStatuses[0].SystemStatus.Details[0].Status',
                        StringEquals: 'passed',
                      },
                      {
                        Variable: '$.InstanceStatuses[0].InstanceStatus.Details[0].Status',
                        StringEquals: 'passed',
                      },
                    ],
                    Next: 'EC2 Pass',
                  },
                ],
                Default: 'EC2 Wait',
              },
              'EC2 Pass': {
                Type: 'Pass',
                End: true,
              },
              'EC2 Wait': {
                Type: 'Wait',
                Seconds: 5,
                Next: 'DescribeInstanceStatus',
              },
            },
          },
        ],
      },
    },
  });
```



For Python, be sure to use a context manager when opening your JSON file.  
- You do not need to `str()` the dictionary object you supply as your `definition` prop.  
- Elements of your override path **do** need to be strings.

```python
secret = Secret(stack, 'Secret')

with open('sample.json', 'r+', encoding='utf-8') as sample:
    sample_dict = json.load(sample)

state_machine = StateMachine(
    self,
    'Test',
    definition = sample_dict,
    overrides = {
    "Read database credentials secret": {
      "Parameters": {
        "SecretId": secret.secret_arn,
      },
    },
  })
```
In this example, the ASL has a state called 'Read database credentials secret' and the SecretId parameter is overridden with a 
CDK generated value.
Future changes can be done by editing, debugging, and testing the state machine directly in the Workflow Studio.
Once everything is working properly, copy and paste the ASL back to your local file.

## Issues

Please open any issues you have on [Github](https://github.com/mbonig/state-machine/issues).

## Contributing

Please submit PRs from forked repositories if you'd like to contribute.