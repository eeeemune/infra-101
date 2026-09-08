# 💚 Terraform: Dependency-Driven Behavior

## 💛 What is it?
Beyond deciding **create order**, Terraform lets a change in one resource **drive behavior** in another: force a replacement, control the create-versus-destroy sequence, ignore drift, or block deletion. These are the `lifecycle` meta-arguments plus a couple of related tools.
Plain version: the dependency graph does not just order actions, it **propagates change**. Replacing one resource can cascade into replacing others, and you get knobs to shape exactly how that cascade behaves.
> This builds on the [Managing Terraform Resource Dependencies] note. That one is about ordering; this one is about what a change does to everything downstream.
## 💛 Why do we need it?
Ordering alone is not enough:
- Sometimes changing A must **recreate** B (an instance must be replaced when its launch template version changes).
- Sometimes you must **create the new before destroying the old** to avoid downtime.
- Sometimes an attribute changes **outside** Terraform and you want it ignored, not fought.
- Sometimes a resource must never be destroyed by accident.
These give you control over the **how and when** of change, not just the order.
## 💛 The lifecycle block
### 🤍 create_before_destroy
By default a replacement is **destroy then create**, which causes downtime. Flip it so the new object is created first.
```hcl
resource "aws_instance" "app" {
  # ...
  lifecycle {
    create_before_destroy = true
  }
}
```
It reorders the graph so the replacement exists before the old one is torn down. It also **propagates**: resources this one depends on may need the same setting, and Terraform will tell you.
### 🤍 prevent_destroy
A guardrail that makes Terraform refuse any plan that would destroy this resource.
```hcl
lifecycle {
  prevent_destroy = true
}
```
Good for a production database. To actually delete it later, you remove the flag first.
### 🤍 ignore_changes
Stop drift on specific attributes from showing up as a diff. Use it for values changed by something other than Terraform (autoscaling adjusting capacity, a tag added by another system).
```hcl
lifecycle {
  ignore_changes = [
    desired_capacity,
    tags["LastModified"],
  ]
}
```
`ignore_changes = all` ignores every attribute (rare, and easy to misuse).
### 🤍 replace_triggered_by (Terraform 1.2+)
Force this resource to be **replaced** when another resource or attribute changes, even if this resource's own config did not change.
```hcl
resource "aws_instance" "app" {
  # ...
  lifecycle {
    replace_triggered_by = [aws_launch_template.app.latest_version]
  }
}
```
The classic use: recreate instances whenever the launch template version updates.
## 💛 How replacement propagates
When Terraform replaces resource A, any resource that reads an attribute of A which changes as a result gets updated or replaced too. An unknown value (`known after apply`) ripples through the graph. That cascade is dependency-driven behavior in action.
```javascript
change launch template version
        |
  replace_triggered_by
        v
replace aws_instance.app   (new id)
        |
  things referencing app.id
        v
update or replace downstream
```
This is why a single upstream replacement can make a plan much larger than expected.
## 💛 Related tools
### 🤍 precondition / postcondition (Terraform 1.2+)
Validate an assumption inside a resource or data lifecycle block, and fail the plan early if a dependency is not what you expected.
```hcl
lifecycle {
  precondition {
    condition     = data.aws_ami.selected.architecture == "x86_64"
    error_message = "Selected AMI must be x86_64."
  }
}
```
### 🤍 terraform_data triggers
Run or replace something based on arbitrary values. `terraform_data` (1.4+) replaces `null_resource` for most uses.
```hcl
resource "terraform_data" "migrate" {
  triggers_replace = [aws_db_instance.main.id]   # re-run when the DB is replaced

  provisioner "local-exec" {
    command = "./run-migration.sh"
  }
}
```
## 💛 Gotcha
- **create_before_destroy needs unique names.** Creating the new object before destroying the old collides if the resource has a fixed name or identifier. Use `name_prefix` or a random suffix so both can exist briefly.
- **create_before_destroy spreads.** Turning it on for one resource can force it onto resources it depends on. Terraform reports this, but expect wider graph effects than the single block suggests.
- **ignore_changes hides real drift too.** You stop seeing every change to that attribute, not just the noisy ones. Scope it to exact attributes and avoid `all`.
- **prevent_destroy blocks legitimate destroys.** You must edit the code to remove it before you can delete the resource or make a change that requires destroy-and-recreate. It can also block a full `terraform destroy`.
- **replace_triggered_by only replaces, and only on references.** It triggers replacement (not an in-place update), and it must point at a resource, one of its attributes, or its `count` / `for_each`, not an arbitrary expression.
- **Known-after-apply cascades.** An upstream replacement can turn many downstream values unknown at plan time, making the plan noisier and forcing more churn than you intended.
## 💛 References
- Terraform: lifecycle meta-arguments: https://developer.hashicorp.com/terraform/language/meta-arguments/lifecycle
- Terraform: custom conditions (precondition / postcondition): https://developer.hashicorp.com/terraform/language/expressions/custom-conditions
- Terraform: the terraform_data resource: https://developer.hashicorp.com/terraform/language/resources/terraform-data
