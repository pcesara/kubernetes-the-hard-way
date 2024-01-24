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

