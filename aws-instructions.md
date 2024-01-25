# AWS instructions

To create the resources in AWS it will require chaining a series of commands. Some of those commands need to pass
as parameters some of the output values of previous commands.

## Create VPC and Subnets

```
aws ec2 create-vpc --cidr-block 10.240.0.0/24 --tag-specifications ResourceType=vpc,Tags='[{Key=Name,Value="k8s-hard-way-vpc"}]'
```

Values to record: VPC ID

```
aws ec2 create-subnet --vpc-id ${VPC_ID} --cidr-block 10.240.0.0/24 --tag-specifications ResourceType=subnet,Tags='[{Key=Name,Value="k8s-hard-way-subnet"}]'
```

Values to record: Subnet ID

## Create firewall and its rules

AWS specifies the use of IANA protocol numbers in their Rule Group definition.
The following protocols are used:
 - ICMP = 1
 - TCP = 6
 - UDP = 17

```
aws network-firewall create-rule-group \
    --rule-group-name k8s-hard-way-rule-group-allow-internal \
    --type STATELESS \
    --capacity 20 \
    --rule-group '{                                                     
    "RulesSource": {                                                    
       "StatelessRulesAndCustomActions": {                              
           "StatelessRules": [                                          
               {                                                        
                   "RuleDefinition": {                                  
                       "MatchAttributes": {                             
                           "Sources": [                                 
                               {                                        
                                   "AddressDefinition": "10.240.0.0/24" 
                               },                                       
                               {                                        
                                   "AddressDefinition": "10.200.0.0/16" 
                               }                                        
                           ],                                           
                           "Protocols": [                               
                              1,                                   
                              6,                                    
                              17                                    
                           ]                                            
                       },                                               
                       "Actions": [                                     
                           "aws:pass"                                   
                       ]                                               
                   },                                                    
                   "Priority": 200                                  
               },
               {                                                        
                   "RuleDefinition": {                                  
                       "MatchAttributes": {                             
                           "Sources": [                                 
                               {                                        
                                   "AddressDefinition": "0.0.0.0/0"     
                               }                                        
                           ],                                           
                           "Protocols": [                               
                              1
                           ]                                            
                       },                                               
                       "Actions": [                                     
                           "aws:pass"                                   
                       ]                                               
                   },                                                    
                   "Priority": 100                                  
               },                                                       
               {                                                        
                   "RuleDefinition": {                                  
                       "MatchAttributes": {                             
                           "Sources": [                                 
                               {                                        
                                   "AddressDefinition": "0.0.0.0/0"     
                               }                                        
                           ],                                           
                           "DestinationPorts": [                        
                              {                                         
                                  "FromPort": 22,                       
                                  "ToPort": 22
                              },                                        
                              {                                         
                                  "FromPort": 6443,                     
                                  "ToPort": 6443                       
                              }                                         
                           ],                                           
                           "Protocols": [                               
                              6                                    
                           ]                                            
                       },                                               
                       "Actions": [                                     
                           "aws:pass"                                   
                       ]                                               
                   },                                                    
                   "Priority": 101                                  
               }                                                        
           ]                                                            
       }                                                                
    }                                                                   
}'
```

Values to record: Rule Group ARN

```
aws network-firewall create-firewall-policy \
    --firewall-policy-name k8s-hard-way-firewall-policy \
    --firewall-policy '{                       
    "StatelessRuleGroupReferences": [          
        {                                      
            "ResourceArn": "${RULE_GROUP_ARN}",
            "Priority": 2                      
        }                                      
    ],                                         
    "StatelessDefaultActions": [               
        "aws:drop"
    ],                                         
    "StatelessFragmentDefaultActions": [       
        "aws:drop"
    ]
}'
```

Values to record: FW Policy ARN

```
aws network-firewall create-firewall \
  --firewall-name k8s-hard-way-firewall \
  --vpc-id ${VPC_ID} \
  --subnet-mappings SubnetId=${SUBNET_ID},"IPV4" \
  --firewall-policy-arn ${FW_POLICY_ARN}
```

## Kubernetes Public IP Address

```
aws ec2 allocate-address \
--region eu-central-1 \
--tag-specification ResourceType=elastic-ip,Tags='[{Key="name",Value="k8s-hard-way-elastic-ip"}]' \
--domain vpc
```

Values to record: Allocation ID

Until the IP is associated with a running instance, AWS will charge an hourly fee.
To reduce spending, it might be better to release the IP address until needed.

```
aws ec2 release-address --allocation-id ${ASSOCIATION_ID}
```

## Compute Instances

### Kubernetes Controllers


    --key-name key-worker${i} \

```
for i in 0 1 2; do
  aws ec2 run-instances \
    --image-id ami-0662ac347458e7d4c \
    --instance-type m6g.large \
    --subnet-id subnet-09d9a327014289c5f \
    --private-ip-address 10.240.0.2${i} \
    --region eu-central-1 \
    --tag-specifications ResourceType=instance,Tags='[{Key="Name",Value="worker-${i}"},{Key="Project",Value="k8s-hard-way"},{Key="Role",Value="Worker"}]' \
    --block-device-mappings '[
    {
        "DeviceName": "/dev/sdh",
        "Ebs": {
            "DeleteOnTermination": true,
            "VolumeSize": 200,
            "VolumeType": "gp2"
        }
    }
]'
done
```

## Deleting resources

```
aws network-firewall delete-firewall --firewall-name k8s-hard-way-firewall
aws network-firewall delete-firewall-policy --firewall-policy-name k8s-hard-way-firewall-policy

```
