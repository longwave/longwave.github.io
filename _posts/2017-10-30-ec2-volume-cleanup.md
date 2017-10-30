---
layout: post
title:  EC2 volume cleanup one liner
---
If you don't have the "delete on termination" flag enabled on your EBS volumes you can end up with a whole load of unused volumes, and over time the cost of this can add up. To clean them all up in one fell swoop:

```
aws ec2 describe-volumes | jq -r '.Volumes[]|select(.State=="available")|.VolumeId' | xargs -n1 -I{} aws ec2 delete-volume --volume-id={}
```
