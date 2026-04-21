# Squid Forward Proxy for Kiro IDE

CloudFormation templates for deploying a Squid forward proxy on AWS. Kiro IDE routes its traffic through the proxy via the `HTTPS_PROXY` environment variable.

Two deployment options are provided:

| Template | Use case | Availability | Cost |
|---|---|---|---|
| `squid-proxy.yaml` | Personal / dev / testing | Single instance, single AZ | ~$8/month (t3.micro) |
| `squid-proxy-ha.yaml` | Production / shared team use | NLB + ASG across 2 AZs | ~$50/month (NLB + NAT + 2x t3.micro) |

## squid-proxy.yaml — Single Instance

A single EC2 instance running Squid in a public subnet with a static Elastic IP.

### Architecture

```
Your laptop ──► Squid EC2 (EIP) ──► Kiro API
                port 3128            port 443
```

### What's included

- VPC with a single public subnet
- EC2 instance (Amazon Linux 2023) with Squid auto-installed
- Elastic IP (static — survives stop/start)
- Security group locked to your IP (port 3128 inbound, port 443 outbound only)
- SSM Session Manager access (no SSH key or port 22 needed)

### Deploy

```bash
# Find your public IP
curl -s https://checkip.amazonaws.com

# Deploy
aws cloudformation deploy \
  --template-file infra/squid-proxy.yaml \
  --stack-name squid-proxy \
  --parameter-overrides AllowedIP="YOUR_IP/32" \
  --capabilities CAPABILITY_IAM \
  --region ap-southeast-1

# Get proxy URL
aws cloudformation describe-stacks \
  --stack-name squid-proxy \
  --region ap-southeast-1 \
  --query 'Stacks[0].Outputs' \
  --output table
```

### Configure Kiro

Set the `ProxyURL` output in Kiro **Settings > Proxy**, or:

```bash
export HTTPS_PROXY=http://<EIP>:3128
```

### Cleanup

```bash
aws cloudformation delete-stack --stack-name squid-proxy --region ap-southeast-1
```

---

## squid-proxy-ha.yaml — High Availability

NLB + Auto Scaling Group with Squid instances in private subnets. Regional NAT Gateway provides outbound internet access.

### Architecture

```
Your laptop ──► NLB (public subnets, port 80) ──► Squid ASG (private subnets, port 3128) ──► Regional NAT GW ──► Kiro API (port 443)
```

### What's included

- VPC with 2 public subnets (NLB) and 2 private subnets (Squid instances)
- Network Load Balancer (internet-facing, TCP port 80)
- Auto Scaling Group (default 2 instances, scales 1–4 based on CPU)
- Regional NAT Gateway (auto-scales across AZs, auto-manages EIPs)
- Squid instances have no public IPs — only reachable via NLB
- Squid egress restricted to HTTPS (port 443) only
- SSM Session Manager access via NAT Gateway
- TCP health checks on port 3128

### Deploy

```bash
curl -s https://checkip.amazonaws.com

aws cloudformation deploy \
  --template-file infra/squid-proxy-ha.yaml \
  --stack-name squid-proxy-ha \
  --parameter-overrides AllowedIP="YOUR_IP/32" \
  --capabilities CAPABILITY_IAM \
  --region ap-southeast-1

aws cloudformation describe-stacks \
  --stack-name squid-proxy-ha \
  --region ap-southeast-1 \
  --query 'Stacks[0].Outputs' \
  --output table
```

### Configure Kiro

```bash
export HTTPS_PROXY=http://<NLB_DNS_NAME>
```

### Cleanup

```bash
aws cloudformation delete-stack --stack-name squid-proxy-ha --region ap-southeast-1
```

---

## Security: How HTTP CONNECT Works

When you set `HTTPS_PROXY=http://...`, the `http://` only describes how the client initiates the proxy handshake. The actual data is encrypted end-to-end.

### The flow

1. Kiro sends a plaintext `CONNECT` request to the proxy:
   ```
   CONNECT q.us-east-1.amazonaws.com:443 HTTP/1.1
   Host: q.us-east-1.amazonaws.com:443
   ```

2. Squid responds:
   ```
   HTTP/1.1 200 Connection Established
   ```

3. After the `200`, the proxy becomes a TCP tunnel. Kiro performs a TLS handshake directly with the Kiro backend through the tunnel. From this point, all traffic is encrypted — Squid only relays opaque bytes.

### What's exposed vs. protected

| Data | Visible to proxy / network? |
|---|---|
| Destination hostname (e.g. `q.us-east-1.amazonaws.com`) | Yes — in the CONNECT line |
| Destination port (`443`) | Yes |
| API request/response bodies | No — encrypted by TLS |
| Auth tokens, message content | No — encrypted by TLS |

The only plaintext information is the destination hostname, which is equivalent to what DNS queries already expose. Squid is configured with `via off` and `forwarded_for delete` to strip proxy-identifying headers.

### What the destination sees

The destination (Kiro API) sees the proxy's IP, not your laptop's IP:

- **Single instance**: destination sees the EC2 Elastic IP
- **HA setup**: destination sees the Regional NAT Gateway's EIP

---

## Optional: Adding TLS to the NLB (HA only)

To encrypt the `CONNECT` handshake between your laptop and the NLB, you can add a TLS listener. This prevents the destination hostname from being visible on the wire between your laptop and the proxy.

### Prerequisites

1. A custom domain name (e.g. `proxy.example.com`) — you cannot use the NLB's default `.elb.amazonaws.com` DNS name for ACM certificates
2. An ACM certificate for that domain in `ap-southeast-1`
3. A DNS record (CNAME or alias) pointing your domain to the NLB DNS name

### Steps

1. **Request a certificate in ACM**

   Go to AWS Certificate Manager in `ap-southeast-1` and request a public certificate for your domain. Complete DNS or email validation.

2. **Add a TLS listener to the NLB**

   In the EC2 console → Load Balancers → select `squid-ha-nlb` → Listeners → Add listener:
   - Protocol: TLS
   - Port: 443
   - Default action: Forward to `squid-ha-tg`
   - Default SSL certificate: select your ACM certificate

3. **Update the NLB security group**

   Add an inbound rule to the NLB security group (output `NLBSecurityGroupId`):
   - Protocol: TCP
   - Port: 443
   - Source: your IP (e.g. `YOUR_IP/32`)

4. **Configure Kiro**

   ```bash
   export HTTPS_PROXY=https://proxy.example.com
   ```

   Note the `https://` — this tells Kiro to TLS-connect to the proxy before sending the `CONNECT` request.

### With TLS enabled

```
Your laptop ──TLS──► NLB (port 443) ──TCP──► Squid (port 3128) ──TLS tunnel──► Kiro API (port 443)
```

The `CONNECT` handshake is now encrypted between your laptop and the NLB. The NLB terminates TLS and forwards plain TCP to Squid. Squid then tunnels the already-encrypted Kiro traffic to the backend.

### Is TLS on the NLB necessary?

For most use cases, no. The actual API payloads are already encrypted end-to-end by TLS inside the CONNECT tunnel. The only additional protection TLS on the NLB provides is hiding the destination hostname in the CONNECT command — which DNS queries already expose anyway. It's a defense-in-depth measure, not a strict requirement.
