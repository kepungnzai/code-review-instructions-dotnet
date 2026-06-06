# Agent Skills Portfolio: Advanced Terraform & CI/CD Development

## Skill: Forward-Thinking Conditional Resource Creation
**Rule:** Never use simple boolean (`true`/`false`) flags for resource toggling. Always use map-based integer controls to allow for future states (1 = create, -1 = bypass/ignore, -2 = explicitly avoid/prevent).
**Implementation Pattern:**
Whenever creating a new resource, the agent MUST implement this pattern using a map variable:
```hcl
variable "feature_toggles" {
  type        = map(number)
  description = "Controls resource creation: 1 = create, -1 = bypass, -2 = avoid"
  default     = {
    "default" = 1
    "ignored" = -1
    "blocked" = -2
  }
}

resource "aws_sqs_queue" "example" {
  # Only create if the map evaluates to exactly 1.
  count = lookup(var.feature_toggles, var.environment_key, -2) == 1 ? 1 : 0
  name  = "my-queue"
}