---
title: "[Terraform] Bug Elasticache replication group security_group_names forces replacement"
date: 2024-04-01 17:38:00 +0900
categories: [TroubleShooting, Terraform]
tags: [aws, terraform, troubleshooting]
---

# 문제 상황
* 콘솔을 통해 프로비저닝 되었던 Elasticache의 Redis Cluster(replication group)를 테라폼으로 import 시킴
* plan 실행시 `security_group_names` 항목에 `forces replacement`가 표시되면서 클러스터를 삭제/생성 시도
* 현재 state 파일에서도 security_group_names는 null이고, plan 할 때에도 null을 넣었지만 상태가 다르다고 인식

```bash
  # module.redis_cluster["purr-ec-redis"].aws_elasticache_replication_group.this[0] must be replaced
-/+ resource "aws_elasticache_replication_group" "this" {
      + apply_immediately              = true
      ~ arn                            = "arn:aws:elasticache:ap-northeast-2:211107293600:replicationgroup:purr-ec-redis" -> (known after apply)
      ~ at_rest_encryption_enabled     = false -> (known after apply)
      ~ auto_minor_version_upgrade     = "true" -> (known after apply)
      ~ cluster_enabled                = false -> (known after apply)
      + configuration_endpoint_address = (known after apply)
      ~ data_tiering_enabled           = false -> (known after apply)
      ~ description                    = " " -> (known after apply)
      ~ engine_version_actual          = "6.2.6" -> (known after apply)
      + global_replication_group_id    = (known after apply)
      ~ id                             = "purr-ec-redis" -> (known after apply)
      ~ maintenance_window             = "sun:16:00-sun:17:00" -> (known after apply)
      ~ member_clusters                = [
          - "purr-ec-redis-001",
        ] -> (known after apply)
      ~ num_node_groups                = 1 -> (known after apply)
      ~ primary_endpoint_address       = "purr-ec-redis.gjfnbb.ng.0001.apn2.cache.amazonaws.com" -> (known after apply)
      ~ reader_endpoint_address        = "purr-ec-redis-ro.gjfnbb.ng.0001.apn2.cache.amazonaws.com" -> (known after apply)
      ~ replicas_per_node_group        = 0 -> (known after apply)
      + security_group_names           = (known after apply) # forces replacement
      ~ snapshot_window                = "15:00-16:00" -> (known after apply)
      ~ transit_encryption_enabled     = false -> (known after apply)
      - user_group_ids                 = [] -> null
        # (12 unchanged attributes hidden)
    }
```

<br>

# 해결 방법

무시가 답😇<br>
ignore_changes에 security_group_names를 추가했더니 no changes
```hcl
lifecycle {  
  ignore_changes = [security_group_names]  
}
```

<br>

Reference
* https://github.com/hashicorp/terraform-provider-aws/issues/32835
