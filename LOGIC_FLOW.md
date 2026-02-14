# Terraform Modules Logic Flow

## Using `public_subnet_az1` in `vpc/main.tf` as example to show the flow and how files are Connected

////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

1. INPUT => `variables.tf`

The resource pulls values from three variables defined in `vpc/variables.tf`:

```JavaScript
variable "public_subnet_az1_cidr" {
  description = "public subnet az1 cidr block"
  type        = string
}

variable "project_name" {
  description = "project name"
  type        = string
}

variable "environment" {
  description = "environment"
  type        = string
}
```

2. LOGIC → `main.tf` (Resource Definition)

The resource uses those variables to build infrastructure:

```JavaScript
resource "aws_subnet" "public_subnet_az1" {
  vpc_id                  = aws_vpc.vpc.id              // References VPC created above in this file
  cidr_block              = var.public_subnet_az1_cidr  // From vpc/variables.tf
  availability_zone       = data.aws_availability_zones.available_zones.names[0]  // From data source
  map_public_ip_on_launch = true                        // Hardcoded: public IPs auto-assigned

  tags = {
    Name = "${var.project_name}-${var.environment}-public-az1"  # ← Interpolates variables
  }
}
```

What this does:

- Gets the VPC ID from `aws_vpc.vpc` created earlier in the same file
- Gets the CIDR block from `variables.tf` (e.g., `10.0.1.0/24`)
- Gets the availability zone dynamically from AWS data source (`eu-west-2`, or second zone)
- Creates tags using project name + environment from variables (e.g., `rentzone-dev-public-az1`)

3. OUTPUT → `outputs.tf`

Every resource created is exported for downstream use:

```JavaScript
output "public_subnet_az1_id" {
  value = aws_subnet.public_subnet_az1.id  // References the subnet created above
}
```

4. CONSUMPTION → `nat-gateway/main.tf`

The NAT Gateway module depends on the VPC module outputs:

```JavaScript
variable "public_subnet_az1_id" {}  // Receives subnet ID from VPC outputs

resource "aws_nat_gateway" "nat_gateway_az1" {
  allocation_id = aws_eip.eip1.id
  subnet_id     = var.public_subnet_az1_id  // Uses the exported subnet ID

  depends_on = [var.internet_gateway]       // Uses VPC IGW exported from outputs
}
```

**How it receives the value:**
When the root config calls this module, it passes the VPC output: `public_subnet_az1_id = module.vpc.public_subnet_az1_id`
This variable then holds that subnet ID and can be used in resources.
