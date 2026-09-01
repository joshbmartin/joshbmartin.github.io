---
title: Connecting to Private EKS Clusters Without CloudShell
date: 2026-03-01 09:00:00 -0500
categories: [devops]
tags: [aws, eks, kubernetes, ssm]
img_path: /assets/img/
---

If you run private EKS clusters — the kind with no public API endpoint — you've probably hit the same wall I did. The Kubernetes API is only reachable from inside the VPC, so you can't just `kubectl get pods` from your laptop. AWS offers a few ways around this, but they all come with friction.

### The CloudShell problem

The default answer from AWS is usually "just use CloudShell" or "use a Cloud9 environment inside the VPC." Both work, technically. But if you've actually tried to do real work in CloudShell, you know the pain:

1. **No persistent tooling** — CloudShell gives you a minimal Amazon Linux environment. Your shell config, aliases, plugins, kubectl plugins, Helm charts — none of it is there. You're starting from scratch every session.
2. **Session timeouts** — Walk away for 20 minutes and your session is gone. Any context you'd built up — environment variables, command history, files you'd pulled down — evaporates.
3. **Limited storage** — You get 1GB of persistent storage in your home directory. That fills up fast if you're working with Helm charts, log files, or anything beyond basic `kubectl` commands.
4. **No local editor integration** — You can't use your IDE. No jumping to a YAML definition in VS Code, no Lens, no k9s running locally. You're stuck in a browser terminal.
5. **Copy-paste workflow** — Need to share a command's output with a teammate? You're selecting text in a browser window and pasting it somewhere else. Need to apply a manifest you've been editing locally? Copy it into CloudShell first.

The fundamental issue is that CloudShell puts *you* inside the VPC, when what you actually want is to bring the *cluster API* to your local machine.

### SSM port forwarding: the better approach

AWS Systems Manager (SSM) Session Manager can create encrypted tunnels from your local machine to resources inside a VPC — no SSH keys, no bastion security groups to maintain, no opening inbound ports. If you already have a bastion host (or any EC2 instance) running the SSM agent in the same VPC as your EKS cluster, you can tunnel the Kubernetes API endpoint through it.

The flow looks like this:

```
laptop:8443 → SSM tunnel → bastion EC2 → EKS API endpoint:443
```

Once the tunnel is up, `kubectl` talks to `localhost:8443`, which SSM forwards through the bastion to the private EKS endpoint. From kubectl's perspective, the cluster is local.

### Automating it with eks-connect

The raw `aws ssm start-session` command is long and requires looking up several values — the bastion instance ID, the EKS API endpoint hostname, your AWS profile, the region. Doing that manually every morning gets old fast.

I wrote a helper script called `eks-connect` that automates the whole flow. Here's what it does:

1. **Selects or prompts for an AWS profile** — uses `fzf` to pick from your configured SSO profiles if you don't specify one
2. **Validates your SSO session** — checks `sts get-caller-identity` and triggers `aws sso login` if the session has expired
3. **Discovers the EKS cluster** — lists clusters in the region via the AWS API and lets you pick one (or auto-selects if there's only one)
4. **Finds the bastion host** — looks up EC2 instances by tag, so you don't have to remember instance IDs
5. **Updates your kubeconfig** — rewrites the cluster's server URL to `https://localhost:<port>` and sets `insecure-skip-tls-verify` (with a backup prompt first)
6. **Starts the SSM tunnel** — opens the port-forwarding session and keeps it running

A typical invocation looks like this:

```bash
# Use a predefined alias with sensible defaults
eks-connect prod

# Or go fully interactive — fzf prompts for everything
eks-connect

# Override specific options
eks-connect --profile my-profile --region eu-central-1 --local-port 9443
```

The script supports cluster aliases for environments you connect to regularly. An alias maps to a default profile, region, and local port number:

```bash
case "$alias" in
  prod)
    DEFAULT_PROFILE="prod-admin"
    DEFAULT_REGION="us-east-1"
    LOCAL_PORT="8443"
    ;;
  dev)
    DEFAULT_PROFILE="dev-admin"
    DEFAULT_REGION="us-east-1"
    LOCAL_PORT="8444"
    ;;
esac
```

Using different local ports per cluster means you can tunnel to multiple clusters simultaneously — `prod` on 8443, `dev` on 8444 — each in its own terminal tab.

### Bastion discovery

Instead of hardcoding instance IDs (which change when instances get recycled), the script discovers bastions dynamically by querying EC2 tags:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Function,Values=eksbastion" \
           "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].[InstanceId,Tags[?Key==`Name`].Value|[0],PrivateIpAddress]' \
  --output text \
  --profile "$profile"
```

If multiple instances match, `fzf` presents them as a list. If there's exactly one, it's selected automatically. This is a small thing that removes a surprising amount of daily friction — you never have to go hunting in the console for the current bastion ID.

### Kubeconfig management

The kubeconfig update is the part that would be easy to mess up manually. The script handles it in three steps:

1. Runs `aws eks update-kubeconfig` to set up the context with proper IAM authentication
2. Overrides the server URL to point at `localhost:<port>`
3. Removes `certificate-authority-data` and sets `insecure-skip-tls-verify: true` (the TLS cert is issued for the real endpoint hostname, not `localhost`)

It also offers to back up your existing kubeconfig before making changes — I've been burned enough times by kubeconfig corruption to appreciate this.

### The workflow

Day-to-day, it works like this:

**Terminal 1** — Start the tunnel:
```bash
eks-connect prod
```

The script validates your SSO session, finds the bastion, updates kubeconfig, and opens the SSM tunnel. It also copies `export AWS_PROFILE=prod-admin` to your clipboard.

**Terminal 2** — Use kubectl normally:
```bash
export AWS_PROFILE=prod-admin
kubectl get nodes
kubectl logs -f deployment/my-app
k9s
```

That's it. You're running `kubectl`, `helm`, `k9s`, or whatever else you want — locally, with your full shell environment, your editor integration, your aliases and plugins. The tunnel runs until you close the terminal or hit Ctrl+C.

### CloudShell vs. SSM tunnel

| | CloudShell | SSM Tunnel |
|---|---|---|
| **Environment** | Minimal Amazon Linux | Your local machine |
| **Tooling** | Start from scratch | Full local setup — aliases, plugins, IDE |
| **Session persistence** | Times out after ~20 min idle | Stays up until you close it |
| **Editor integration** | Browser terminal only | VS Code, Lens, k9s, whatever you use |
| **Multi-cluster** | One session at a time | Multiple tunnels on different ports |
| **Prerequisites** | Just a browser | AWS CLI, SSM plugin, bastion host in VPC |
| **Network requirements** | None (runs in AWS) | Outbound HTTPS (SSM uses port 443) |
| **Auditability** | CloudTrail | CloudTrail + SSM session logs |

The one area where CloudShell wins is zero setup — you open a browser and you're in. The SSM approach requires a bastion host and some one-time configuration. But if you're connecting to EKS daily, the productivity difference is massive.

### Prerequisites and setup

To use this approach, you need:

- **AWS CLI v2** — `brew install awscli`
- **Session Manager plugin** — `brew install --cask session-manager-plugin`
- **kubectl** — `brew install kubectl`
- **fzf** — `brew install fzf` (for the interactive prompts)
- **jq** — `brew install jq`
- **A bastion host** — Any EC2 instance in the VPC running the SSM agent. Tag it with something identifiable so the script can find it.
- **SSO profiles configured** — `aws configure sso --profile <profile-name>`

One-time kubeconfig setup is handled by the script itself on first run.

### Wrapping up

Private EKS clusters don't have to mean working inside a constrained browser terminal. An SSM tunnel through a bastion host brings the cluster API to your local machine, where your real tools live. The `eks-connect` script wraps the tedious parts — SSO validation, bastion discovery, kubeconfig management, tunnel setup — into a single command.

If you're on a team that uses private EKS clusters and you haven't tried this approach, give it a shot. The combination of SSM port forwarding + a little automation makes the experience feel like the cluster is running locally.
