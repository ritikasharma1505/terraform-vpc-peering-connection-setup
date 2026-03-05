# VPC PEERING

resource "aws_vpc_peering_connection" "primary_to_secondary" {
  provider    = aws.primary
  vpc_id      = aws_vpc.primary_vpc.id
  peer_vpc_id = aws_vpc.secondary_vpc.id
  peer_region = var.secondary_region
  auto_accept = false

  tags = {
    Name = "Cross-Region-Peering"
  }
}

resource "aws_vpc_peering_connection_accepter" "secondary_accepter" {
  provider                  = aws.secondary
  vpc_peering_connection_id = aws_vpc_peering_connection.primary_to_secondary.id
  auto_accept               = true
}

# ENABLE DNS RESOLUTION

resource "aws_vpc_peering_connection_options" "primary_dns" {
  provider                  = aws.primary
  vpc_peering_connection_id = aws_vpc_peering_connection.primary_to_secondary.id

  requester {
    allow_remote_vpc_dns_resolution = true
  }

  depends_on = [aws_vpc_peering_connection_accepter.secondary_accepter]
}

resource "aws_vpc_peering_connection_options" "secondary_dns" {
  provider                  = aws.secondary
  vpc_peering_connection_id = aws_vpc_peering_connection.primary_to_secondary.id

  accepter {
    allow_remote_vpc_dns_resolution = true
  }

  depends_on = [aws_vpc_peering_connection_accepter.secondary_accepter]
}

# ROUTES FOR PEERING

resource "aws_route" "primary_to_secondary" {
  provider                  = aws.primary
  route_table_id            = aws_route_table.primary_rt.id
  destination_cidr_block    = var.secondary_vpc_cidr
  vpc_peering_connection_id = aws_vpc_peering_connection.primary_to_secondary.id

  depends_on = [
    aws_vpc_peering_connection.primary_to_secondary,
    aws_vpc_peering_connection_accepter.secondary_accepter
  ]
}

resource "aws_route" "secondary_to_primary" {
  provider                  = aws.secondary
  route_table_id            = aws_route_table.secondary_rt.id
  destination_cidr_block    = var.primary_vpc_cidr
  vpc_peering_connection_id = aws_vpc_peering_connection.primary_to_secondary.id

  depends_on = [
    aws_vpc_peering_connection.primary_to_secondary,
    aws_vpc_peering_connection_accepter.secondary_accepter
  ]
}