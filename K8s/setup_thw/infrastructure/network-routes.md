# Configure the network routes for the cluster

create the route in the route table

worker 0
```
  aws ec2 create-route \
    --route-table-id "${ROUTE_TABLE_ID}" \
    --destination-cidr-block "10.200.0.0/24" \
    --instance-id "	i-0c7099db4a95e397b"
```

worker 1
```
  aws ec2 create-route \
    --route-table-id "${ROUTE_TABLE_ID}" \
    --destination-cidr-block "10.200.1.0/24" \
    --instance-id "	i-082fffee815d71350"
```

worker 2
```
  aws ec2 create-route \
    --route-table-id "${ROUTE_TABLE_ID}" \
    --destination-cidr-block "10.200.2.0/24" \
    --instance-id "	i-00ea71c462c31d2dd"
```