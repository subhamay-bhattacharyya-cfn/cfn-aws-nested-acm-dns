# CloudFormation Nested Stack Template : AWS ACM with DNS

<!-- Row 1: Status - Most Important -->
[![Release](https://img.shields.io/github/v/release/subhamay-bhattacharyya-cfn/cfn-aws-nested-acm-dns?label=Release)](https://github.com/subhamay-bhattacharyya-cfn/cfn-aws-nested-acm-dns/releases)&nbsp;[![Release Workflow](https://github.com/subhamay-bhattacharyya-cfn/cfn-aws-nested-acm-dns/actions/workflows/ci.yaml/badge.svg)](https://github.com/subhamay-bhattacharyya-cfn/cfn-aws-nested-acm-dns/actions/workflows/ci.yaml)&nbsp;[![Issues](https://img.shields.io/github/issues/subhamay-bhattacharyya-cfn/cfn-aws-nested-acm-dns)](https://github.com/subhamay-bhattacharyya-cfn/cfn-aws-nested-acm-dns/issues)&nbsp;[![Last Commit](https://img.shields.io/github/last-commit/subhamay-bhattacharyya-cfn/cfn-aws-nested-acm-dns)](https://github.com/subhamay-bhattacharyya-cfn/cfn-aws-nested-acm-dns/commits)

<!-- Row 2: Code Quality -->
[![Top Language](https://img.shields.io/badge/Languages-YAML-blue)](https://github.com/subhamay-bhattacharyya-cfn/cfn-aws-nested-acm-dns)&nbsp;[![Commits](https://img.shields.io/github/commit-activity/t/subhamay-bhattacharyya-cfn/cfn-aws-nested-acm-dns)](https://github.com/subhamay-bhattacharyya-cfn/cfn-aws-nested-acm-dns/commits)

<!-- Row 3: Tech Stack -->
[![CloudFormation](https://img.shields.io/badge/CloudFormation-IaC-orange?logo=amazonaws&logoColor=white)](https://aws.amazon.com/cloudformation/)&nbsp;[![Built with Claude Code](https://img.shields.io/badge/Built_with-Claude_Code-D97757?logo=anthropic&logoColor=white)](https://claude.ai/)

<!-- Row 4: Repository Info -->
[![Files](https://img.shields.io/github/directory-file-count/subhamay-bhattacharyya-cfn/cfn-aws-nested-acm-dns)](https://github.com/subhamay-bhattacharyya-cfn/cfn-aws-nested-acm-dns)&nbsp;[![Repo Size](https://img.shields.io/github/repo-size/subhamay-bhattacharyya-cfn/cfn-aws-nested-acm-dns)](https://github.com/subhamay-bhattacharyya-cfn/cfn-aws-nested-acm-dns)&nbsp;[![Release Date](https://img.shields.io/github/release-date/subhamay-bhattacharyya-cfn/cfn-aws-nested-acm-dns)](https://github.com/subhamay-bhattacharyya-cfn/cfn-aws-nested-acm-dns/releases)

<!-- Row 5: Custom Metrics -->
[![Custom Endpoint](https://img.shields.io/endpoint?url=https://gist.githubusercontent.com/bsubhamay/76a395931a9900b063233df987207cb5/raw/cfn-aws-nested-acm-dns.json)](https://gist.github.com/subhamay-bhattacharyya-cfn/76a395931a9900b063233df987207cb5)

## ACM Certificate with DNS

`cloudformation/template.yaml` creates an AWS Certificate Manager (ACM) public certificate validated through DNS in an existing Route 53 hosted zone. It also creates a Route 53 record that points a domain name at a target such as an Application Load Balancer, CloudFront distribution, or any hostname.

**Resources created:**

- `AWS::CertificateManager::Certificate`: public certificate for `DomainName`, with DNS validation and Certificate Transparency logging enabled. Tagged `Name` as `{ProjectName}-{DomainName}-{Environment}`, with `-{CiSuffix}` appended when `CiSuffix` is set.
- `AWS::Route53::RecordSet`: a `CNAME` or `A` record for `DomainName`

**Parameters:**

Parameters are grouped in the CloudFormation console using the `AWS::CloudFormation::Interface` metadata.

| Group | Parameter | Description |
| ----- | --------- | ----------- |
| Project | `ProjectName` | Project prefix used in the certificate `Name` tag (required) |
| Project | `Environment` | Environment label used in the `Name` tag, e.g. `devl`, `stag`, `prod` (required) |
| Project | `CiSuffix` | Optional suffix for CI/CD unique deployments (default `""`) |
| Certificate | `DomainName` | Fully qualified domain name, e.g. `app.example.com`. A leading `*.` is allowed for wildcard certificates. |
| Certificate | `SubjectAlternativeNames` | Optional comma-separated extra names for the certificate |
| Certificate | `ValidationMethod` | Certificate validation method (default `DNS`, the only supported value) |
| DNS record | `HostedZoneId` | ID of the existing Route 53 public hosted zone |
| DNS record | `RecordType` | `CNAME` (default) or `A` |
| DNS record | `DnsTargetValue` | Hostname for `CNAME`, or IPv4 address for `A` |
| DNS record | `DnsTTL` | TTL in seconds for the record (default `300`, applies to `CNAME` only) |

**Outputs:** `CertificateArn` (exported as `<stack-name>-CertificateArn`), `DomainName`, and `DnsRecordName`.

**Example deployment:**

```bash
aws cloudformation deploy \
  --template-file cloudformation/template.yaml \
  --stack-name acm-app-dev \
  --parameter-overrides \
    ProjectName=cfn-nstd \
    Environment=devl \
    HostedZoneId=Z0123456789ABCDEF \
    DomainName=app.example.com \
    DnsTargetValue=my-alb-123.us-east-1.elb.amazonaws.com \
  --region us-east-1
```

> **Note:** DNS validation records are created only for `DomainName`. Names listed in `SubjectAlternativeNames` will not be validated automatically and the stack will wait on them until they are validated.
>
> CloudFront requires certificates in `us-east-1`.

## Best Practices Implemented

- ✅ DNS validation through Route 53, so renewals need no manual steps
- ✅ Certificate Transparency logging enabled
- ✅ Parameters grouped into Project, Certificate, and DNS record sections in the console
- ✅ Export values for cross-stack references
- ✅ Conditional settings: TTL only for `CNAME`, SANs only when provided, CI suffix only when set

## License

MIT
