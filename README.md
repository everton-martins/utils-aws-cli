# aws-cli


## RDS Estatisticas de conexões


	aws --region sa-east-1 rds describe-db-instances |sed -e 's/^[ \t]*//' \
	  |grep ^\"DBInstanceIdentifier\" \
  	| grep -E 'hom|hlg|hml|dev|dsv' \
  	| awk -F " " '{print $2}'| tr -d '"' \
  	|while read DB; do \
  	printf "$DB\n" && \
  	aws --region sa-east-1 cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name DatabaseConnections --start-time `date -u '+%FT%TZ' -d '2 months ago'` --end-time `date -u '+%FT%TZ'` --statistics Average --period 3600 --dimensions Name=DBInstanceIdentifier,Value=$DB \
  	|jq '.Datapoints[0] | .Average' ; \
	done



## Describe instance EC2

	aws ec2 describe-instances --instance-ids "i-00ae0b338c2e1eed2" --query 'Reservations[].Instances[].EnaSupport'


## Stop Instance

	aws ec2 stop-instances --instance-ids $1

## Detach volume

	aws ec2 detach-volume --volume-id $1 --force


## DYNAMODB


### Describe table scaling

	aws --region us-east-1 application-autoscaling describe-scalable-targets --service-namespace dynamodb --resource-id "table/TABLE_XYZ"


### Describe table scaling policy
	
	aws --region us-east-1 application-autoscaling describe-scaling-policies --service-namespace dynamodb --resource-id "table/TABLE_XYZ" --policy-name "AWSServiceRoleForApplicationAutoScaling_DynamoDBTable"


### Describe table capacity

	aws --region us-east-1 dynamodb describe-table --table-name WALLET_TRANSACTION --query "Table.[TableName,TableStatus,ProvisionedThroughput]"


### Register autoScaling

	aws --region us-east-1 application-autoscaling register-scalable-target --service-namespace dynamodb --resource-id "table/WAF-Lambda-Cloudfront" --scalable-dimension "dynamodb:table:WriteCapacityUnits" --min-capacity 1 --max-capacity 10

	aws --region us-east-1 application-autoscaling register-scalable-target --service-namespace dynamodb --resource-id "table/WAF-Lambda-Cloudfront" --scalable-dimension "dynamodb:table:ReadCapacityUnits" --min-capacity 1 --max-capacity 10


### Put Scaling

	aws --region us-east-1 application-autoscaling put-scaling-policy --service-namespace dynamodb --resource-id "table/WAF-Lambda-Cloudfront" --scalable-dimension "dynamodb:table:WriteCapacityUnits" --policy-name "MyScalingPolicy" --policy-type "TargetTrackingScaling" --target-tracking-scaling-policy-configuration file://DBWrite-scaling-policy.json

	aws --region us-east-1 application-autoscaling put-scaling-policy --service-namespace dynamodb --resource-id "table/WAF-Lambda-Cloudfront" --scalable-dimension "dynamodb:table:ReadCapacityUnits" --policy-name "MyScalingPolicy" --policy-type "TargetTrackingScaling" --target-tracking-scaling-policy-configuration file://DBRead-scaling-policy.json



## Get limits information


	awslimitchecker -W 97 --critical=98 --no-color


## ADD SECURITY GROUP TO RDSs


### Ligar todos que estão desligados
	--salvar os ids em um arquivo
	
		aws rds describe-db-instances --query 'DBInstances[?DBInstanceStatus==`stopped` && DBSubnetGroup.DBSubnetGroupName==`awp-dev-db`][DBInstanceIdentifier]' --output text > ids-stoppeds.txt

	--loop no arquivo e ligar

		cat ids-stoppeds.txt | while read LINE; do aws rds start-db-instance --db-instance-identifier $LINE; done


## altera todos os da rede de DEV

		aws rds describe-db-instances --query 'DBInstances[?DBSubnetGroup.DBSubnetGroupName==`awp-dev-db`][DBInstanceIdentifier,join(`,`,VpcSecurityGroups[*].VpcSecurityGroupId)]' --output text | while read LINE; do $(echo "aws rds modify-db-instance --db-instance-identifier "$(echo $LINE | cut -d" " -f1)" --vpc-security-group-ids "$(echo $LINE | cut -d" " -f2 | tr "," " ")" sg-05630ff77a983a8b4 --apply-immediately"); done


## Desligar todos que foram ligados
	--loop no arquivo e desligar

		cat ids-stoppeds.txt | while read LINE; do aws rds stop-db-instance --db-instance-identifier $LINE; done


## REGISTER DEV DATABASES ON "AGIBANK-DEV.IN"

	-- Fazer com jq

	aws rds describe-db-instances --query 'DBInstances[?DBSubnetGroup.DBSubnetGroupName==`awp-dev-db`].Endpoint.Address' --output table \
  	| grep "|" | cut -d"|" -f2 | cut -c3- \
  	| while read LINE; \
       do echo -ne "aws route53 change-resource-record-sets --hosted-zone-id Z30N255VLTE3FX --change-batch '{ \"Comment\": \"Automation\", \"Changes\": [ { \"Action\": \"UPSERT\", \"ResourceRecordSet\": "; \
	      CNAME=$(aws route53 list-resource-record-sets --hosted-zone-id Z38ERPC71LLE7 --query "ResourceRecordSets[?ResourceRecords[?Value=='$LINE']]"); echo ${CNAME::-1}" }]}' " | cut -c2- | sed -e 's/company.aws.local/company-dev.in/g'; \
	   done \
  	| grep -v ":  }]}"


## MULTI ACCOUNT

	~/.aws/config

[default]
output = json
region = sa-east-1
[profile dev]
region = us-east-1
role_arn = arn:aws:iam::9999999999999:role/db_automation   #Role of another account
credential_source=Ec2InstanceMetadata			  #Role of EC2 Instance


Na conta de origem editar a relação de confiança na Role da EC2 acima:

{  
   "Version":"2012-10-17",
   "Statement":[  
      {  
         "Effect":"Allow",
         "Principal":{  
            "Service":"ec2.amazonaws.com"
         },
         "Action":"sts:AssumeRole"
      },
      {  
         "Effect":"Allow",
         "Principal":{  
            "AWS":"arn:aws:iam::99999999999999:role/db_automation"
         },
         "Action":"sts:AssumeRole"
      }
   ]
}



## INSTALL AWS CLI

`curl -O https://bootstrap.pypa.io/get-pip.py`
`python get-pip.py `
