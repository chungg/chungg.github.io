---
layout: post
title:  "minimal downtime rds restoration in terraform"
date:   2022-02-11 00:00:00 -0500
tags: aws devops terraform
---

**Disclaimer: i've no idea if this is best practice**

A critical part of testing whether non-production code works is to test it against the actual
production data. This way tests can be done to ensure migrations work and any cases not caught by
test data can be discovered. It can be done by backing up, deploying the code against
the actual production database, and hoping for the best or that no user notices a rollback/restore
(if you're the type of person who enjoys mainlining things). More commonly, alternate environments
or accounts with similar configurations are available and all that is needed is a copy of
the production data.

There are many ways to copy and restore a database, a common technique is to use snapshots and this
functionality is available in RDS. That said, restoring from a snapshot in RDS involves creating
an entirely new cluster which itself takes over 15 mins and can take significantly longer depending
on the database size.

If your database is managed by Terraform and you simply rename and set a `snapshot_identifier`
to the existing resource, this will first trigger a destroy action on the existing database and
only then will it start building the cluster based on the snapshot. This can result in hours where
the database is not accessible to developers.

To continue to manage the resource in Terraform and avoid downtime, the following template can be
used:

```terraform
variable "cluster_map" {
  type        = map(string)
  default     = {one = null, two = "snapshot-name"}
}

variable "active_cluster" {
  type        = string
  default     = "two"
}

resource "aws_db_parameter_group" "this" {
  name        = "${var.name}-aurora-db-postgres13-parameter-group"
  family      = "aurora-postgresql13"
  description = "${var.name}-aurora-db-postgres13-parameter-group"
  tags        = local.tags
  lifecycle {
    create_before_destroy = true
  }
}
resource "aws_rds_cluster_parameter_group" "this" {
  name        = "${var.name}-aurora-postgres13-cluster-parameter-group"
  family      = "aurora-postgresql13"
  description = "${var.name}-aurora-postgres13-cluster-parameter-group"
  tags        = local.tags
  lifecycle {
    create_before_destroy = true
  }
}

module "rds-aurora" {
  source  = "terraform-aws-modules/rds-aurora/aws"
  version = "6.1.3"

  for_each = var.cluster_map

  name           = each.key
  engine         = "aurora-postgresql"
  engine_version = "13"
  instance_class = var.rds_size
  instances = {
    one   = {}
  }
  port = var.port
  master_password = var.pw

  snapshot_identifier = each.value

  vpc_id                      = var.vpc_id
  subnets                     = var.db_subnets
  create_security_group       = true
  allowed_cidr_blocks         = var.db_blocks
  security_group_egress_rules = {
    to_cidrs = {
      cidr_blocks = var.db_blocks
      description = "Egress for private subnets"
    }
  }
  db_parameter_group_name               = aws_db_parameter_group.this.id
  db_cluster_parameter_group_name       = aws_rds_cluster_parameter_group.this.id
  performance_insights_enabled          = var.performance_insights_enabled
  performance_insights_retention_period = var.performance_insights_retention_period
  enabled_cloudwatch_logs_exports       = ["postgresql"]

  skip_final_snapshot       = var.skip_final_snapshot
  copy_tags_to_snapshot     = var.copy_tags_to_snapshot
  tags = local.tags
}

resource "aws_route53_record" "db" {
  zone_id = data.aws_route53_zone.private.zone_id
  name    = "${var.name}-writer"
  type    = "CNAME"
  ttl     = "300"
  records = [module.rds-aurora[var.active_cluster].cluster_endpoint]
}

resource "aws_route53_record" "db-reader" {
  zone_id = data.aws_route53_zone.private.zone_id
  name    = "${var.name}-reader"
  type    = "CNAME"
  ttl     = "300"
  records = [module.rds-aurora[var.active_cluster].cluster_reader_endpoint]
}
```

There are many input variables missing above but essentially, it leverages `for_each`
functionality and allows the ability to specify the clusters that are deployed. It also uses
Route53 names to decide which cluster is accessible. The key variables would be `var.cluster_map`,
`var.active_clsuter`, and `var.pw`. `var.pw` is not critical but the module will end up
creating a new password each time so it can get annoying having to lookup new password each time.

The workflow is as follows:

1. At time T-1, `var.cluster_map` is `{db-a = null}`, where `db-a` is the cluster name and
   `null` means we're creating it as empty cluster initially. `var.active_cluster` is set to 
   `db-a` as it is the only cluster. Our DNS route will point to `db-a`.
2. At time T, we want to test our production snapshot (this part is done outside of Terraform).
   Using the snapshot we've created (and shared if necessary), we set `var.cluster_map` to
   `{db-a = null, db-b = 'db-snapshot-name'}` and `var.active_cluster` to `db-b`. This will 
   create a new cluster based on the snapshot and when it's ready, our DNS route will switch to
   it and leave the initial cluster dangling.
3. At time T+1, we set `var.cluster_map` to `{db-b = 'db-snapshot-name'}` and this will cause
   the initial cluster to be destroyed.

Using this process, while it may still take hours to actually restore the snapshot, the db can be
managed in Terraform (minus the snapshot creation) and manual intervention isn't needed to switch
to the restored snapshot when ready. Additionally, there is an auditable trail of when things
happened as oppose to attempting to find who ran `aws rds restore-db-cluster-from-snapshot` 
somewhere and with what snapshot.
