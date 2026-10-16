# AWS VPC with Public/Private Subnets, NAT Gateway and Bastion Access

A custom VPC in `eu-west-2` (London) with a public and a private subnet. The public subnet reaches the internet through an Internet Gateway. The private subnet reaches the internet outbound-only through a NAT Gateway. Two EC2 instances demonstrate the setup, and security groups restrict who can connect to them.

## Architecture

```
                    Internet
                       |
              [ Internet Gateway ]
                       |
   +-------------------+--------------------------------+
   |  VPC  10.0.0.0/16                                   |
   |                                                     |
   |  Public subnet 10.0.1.0/24   (route: 0.0.0.0/0 -> IGW)
   |    - Public EC2 (also the bastion host)             |
   |    - NAT Gateway + Elastic IP                       |
   |                                                     |
   |  Private subnet 10.0.2.0/24  (route: 0.0.0.0/0 -> NAT)
   |    - Private EC2 (no public IP)                     |
   +-----------------------------------------------------+
```

| Resource | Name | Value |
|---|---|---|
| VPC | `my-vpc` | `10.0.0.0/16` |
| Public subnet | `public-subnet` | `10.0.1.0/24`, `eu-west-2a` |
| Private subnet | `private-subnet` | `10.0.2.0/24`, `eu-west-2a` |
| Internet Gateway | `my-igw` | attached to `my-vpc` |
| Elastic IP | `nat-eip` | allocated for the NAT Gateway |
| NAT Gateway | `my-nat` | in `public-subnet` |
| Route tables | `public-rt`, `private-rt` | see section 3 |
| Security groups | `public-sg`, `private-sg` | see section 4 |
| EC2 instances | `public-ec2`, `private-ec2` | Amazon Linux 2023 |

---

## 1. Create the VPC

Created a custom VPC with CIDR `10.0.0.0/16`, DNS hostnames enabled, then two `/24` subnets in the same Availability Zone. Auto-assign public IPv4 is enabled on the public subnet only.

**VPC**

![VPC details](Screenshots/01-vpc.png)

**Public subnet** (`10.0.1.0/24`, auto-assign public IPv4: Yes)

![Public subnet](Screenshots/02-public-subnet.png)

**Private subnet** (`10.0.2.0/24`, auto-assign public IPv4: No)

![Private subnet](Screenshots/03-private-subnet.png)

---

## 2. Internet Access

**Internet Gateway**: created `my-igw` and attached it to the VPC. This is the two-way door between the VPC and the internet.

![Internet Gateway attached](Screenshots/04-my-igw.png)

**Elastic IP**: allocated a static public IP (`nat-eip`) for the NAT Gateway to use.

![Elastic IP](Screenshots/05-eip.png)

**NAT Gateway**: created `my-nat` in the **public** subnet using the Elastic IP. It sits in the public subnet because it needs the IGW route itself. It lets private instances start outbound connections without being reachable from the internet.

![NAT Gateway available](Screenshots/06-my-nat.png)

---

## 3. Route Tables

| Route table | Destination | Target | Associated subnet |
|---|---|---|---|
| `public-rt` | `0.0.0.0/0` | Internet Gateway (`my-igw`) | `public-subnet` |
| `public-rt` | `10.0.0.0/16` | local | |
| `private-rt` | `0.0.0.0/0` | NAT Gateway (`my-nat`) | `private-subnet` |
| `private-rt` | `10.0.0.0/16` | local | |

The route table, not the subnet's name, is what makes a subnet public or private.

**Public route table: routes**

![Public route table routes](Screenshots/07-public-rt-routes.png)

**Public route table: subnet association**

![Public route table association](Screenshots/08-public-rt-subnet-assoc.png)

**Private route table: routes**

![Private route table routes](Screenshots/09-private-rt-routes.png)

**Private route table: subnet association**

![Private route table association](Screenshots/10-private-rt-subnet-assoc.png)

---

## 4. Security Groups

**`public-sg`** (public EC2)

| Type | Port | Source |
|---|---|---|
| SSH | 22 | my IP only (`/32`) |
| HTTP | 80 | my IP only (`/32`) |

![Public security group](Screenshots/11-public-sg.png)

**`private-sg`** (private EC2)

| Type | Port | Source |
|---|---|---|
| SSH | 22 | `public-sg` (security group reference) |

Outbound rules are left at the default (allow all) so the private instance can reach the internet through the NAT Gateway for updates.

![Private security group](Screenshots/12-private-sg.png)

**Why these rules**
- SSH is never opened to `0.0.0.0/0`, since bots constantly scan port 22.
- `private-sg` references `public-sg` instead of an IP range, so only instances carrying that group can connect, even if their IPs change.
- Security groups are stateful, so reply traffic is allowed automatically.

---

## 5. EC2 Instances

| Instance | Subnet | Public IP | Security group |
|---|---|---|---|
| `public-ec2` | `public-subnet` | Yes | `public-sg` |
| `private-ec2` | `private-subnet` | No | `private-sg` |

**Public EC2 networking**

![Public EC2 networking](Screenshots/13-public-ec2-network.png)

**Private EC2 networking**: only a private `10.0.2.x` address, no public IPv4.

![Private EC2 networking](Screenshots/14-private-ec2-network.png)

**Both instances running**

![Instances running](Screenshots/15-instances-running.png)

---

## 6. Verification

### 6.1 SSH into the public EC2

```bash
ssh -i my-key.pem ec2-user@<PUBLIC_EC2_PUBLIC_IP>
```

![SSH to public EC2](Screenshots/16-ssh-public.png)

The public EC2 also runs a simple web server (installed via user data), reachable on HTTP only from my IP:

![Web page served by the public EC2](Screenshots/17-ssh-public-webpage.png)

### 6.2 Reach the private EC2 through the bastion (jump host)

The public EC2 acts as the bastion host. The key stays on my laptop and is never copied to the server.

**Troubleshooting: permission denied on the jump host**

My first attempt failed:

```bash
ssh -i my-key.pem -J ec2-user@<PUBLIC_EC2_PUBLIC_IP> ec2-user@<PRIVATE_EC2_PRIVATE_IP>
# Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
```

![Permission denied on jump host](Screenshots/17a-ssh-jump-permission-denied.png)

**Cause:** the `-i` option only applies to the final destination, not to the jump host, so the bastion rejected the connection.

**Fix:** load the key into the SSH agent so it is offered to every hop, then connect without `-i`:

```bash
ssh-add my-key.pem
ssh -J ec2-user@<PUBLIC_EC2_PUBLIC_IP> ec2-user@<PRIVATE_EC2_PRIVATE_IP>
```

![SSH to private EC2 via bastion](Screenshots/17b-ssh-private-success.png)

The prompt shows `ip-10-0-2-15` (inside the private subnet), and the "Last login ... from 10.0.1.66" line shows the connection arrived from the bastion's private IP.

### 6.3 Private EC2 reaches the internet through the NAT Gateway

From the private instance:

```bash
curl -s https://checkip.amazonaws.com
```

The address returned is the NAT Gateway's Elastic IP, proving outbound traffic leaves via the NAT even though the instance has no public IP of its own.

![Outbound via NAT](Screenshots/18-nat-test.png)

### 6.4 Private EC2 is not reachable directly from the internet

From my laptop, with no jump host:

```bash
ssh -o ConnectTimeout=10 ec2-user@<PRIVATE_EC2_PRIVATE_IP>
```

The connection times out because the private address has no route from the internet, which confirms the instance is isolated.

![Direct access to private EC2 times out](Screenshots/19-private-unreachable.png)

---

## Bonus

Bastion host: the public EC2 doubles as the bastion. `private-sg` only accepts SSH from `public-sg`, so the private instance can only be reached through it.
---

## Cleanup

NAT Gateways and Elastic IPs are billed while they exist, so everything was removed afterwards in this order:

1. Terminated both EC2 instances
2. Deleted the NAT Gateway (waited for *Deleted*)
3. Released the Elastic IP
4. Detached and deleted the Internet Gateway
5. Deleted the subnets, custom route tables and security groups
6. Deleted the VPC

---

## What I Learned

- A subnet is public or private because of its route table, not its name.
- A NAT Gateway gives private instances outbound-only internet access, and it must itself live in a public subnet.
- Referencing a security group as a source is tighter and more robust than allowing an IP range.
- SSH's `-i` flag does not apply to the `-J` jump host, so the SSH agent is the cleanest way to authenticate both hops without copying private keys to the bastion.
