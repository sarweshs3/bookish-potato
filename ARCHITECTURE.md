# Architecture Design

## 1. Purpose and scope

This repository provisions and configures a Rocky Linux 9 host for a Node.js application. It separates operating-system preparation, security controls, application runtime, deployment, reverse proxying, and operational maintenance into reusable Ansible roles.

I had written the playbooks separately for each task as I did not know about roles in Ansible. I came to understand the difficulty first hand to configure each and every playbook manually.

## 2. Why the playbooks use roles

Ansible roles are used instead of placing every task in one long playbook because each operational responsibility has a clear boundary.

This structure keeps `site.yml` focused on orchestration rather than implementation details. A role can be reviewed, tested, or reused without navigating an unrelated set of tasks. Role-specific templates and handlers live beside the tasks that use them, which makes changes easier to locate.


## 3. Security and operational choices

### Least privilege

As the Node.js process should run as a dedicated, non-root system user. SSH access, firewall rules, and service permissions should be kept as restrictive as the deployment requirements permit. Only the ports needed for administration and web traffic should be exposed.

### Secrets and configuration

Keep non-secret configuration in inventory variables and templates. Use Ansible Vault for encrypted variables when using this repository directly, or integrate a managed secrets service when moving to a larger delivery platform. Restrict access to decrypted values and avoid logging secrets.

### Change safety

Review firewall changes carefully to avoid locking out administrators.

### Health and logs

The health-check role periodically requests the local application health endpoint, records a timestamp and status code, and can send an email alert for non-200 responses. The target host must have a working mail transport and the alert recipient must be configured. Log rotation prevents the application logs from growing without bound. Monitoring should also cover service availability, host resource pressure, deployment failures, and alert delivery.


## 4. What I would use for five engineers deploying ten times a day

At that frequency, the main risks are inconsistent manual execution, deploying untested changes, rebuilding different artifacts for different environments, and slow or uncertain recovery. 

## 5. Safe operating checklist

Before using this repository against a real host:

- Review inventory and confirm the intended target hosts.
- Set the correct application repository, branch, runtime variables, domain, and alert email.
- Configure secrets without committing plaintext credentials.
- Confirm SSH key access and an out-of-band recovery route before hardening.
- Verify mail transport if health-check email alerts are expected.
- Run syntax checks and linting, then test on a disposable/staging Rocky Linux 9 host.
- Review the planned changes and confirm the firewall and service ports.
- Apply to production only after validation and an appropriate change review.
