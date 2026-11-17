# Day 02 - Linux Access Control & SSH Security

## Objective

Practice Linux access control, SSH security configuration,
firewall policies, and authentication log analysis.

## Lab Environment

- Host: macOS / Apple Silicon
- Virtualization: UTM
- Guest: Ubuntu Server ARM64

## 1. User & Permission Management

- [ ] Inspect current user and groups
- [ ] Create securitylab test account
- [ ] Verify sudo privileges
- [ ] Create a restricted test file
- [ ] Verify file permission 600

## 2. SSH Security

- [ ] Check OpenSSH server installation
- [ ] Back up sshd_config
- [ ] Inspect effective SSH configuration
- [ ] Identify root login and authentication policies

## 3. Firewall

- [ ] Inspect UFW status
- [ ] Configure default incoming/outgoing policies
- [ ] Allow SSH traffic
- [ ] Verify firewall rules

## 4. Authentication Log Analysis

- [ ] Inspect SSH service logs
- [ ] Review login history
- [ ] Review failed login attempts
- [ ] Identify authentication failure patterns

## 5. Network & Service Inspection

- [ ] Check SSH service status
- [ ] Inspect listening ports
- [ ] Identify the process listening on port 22

## Findings

### User & Permission Findings

Record actual command outputs and observations here.

### SSH Configuration Findings

Record effective SSH settings here.

### Firewall Findings

Record active UFW policies and rules here.

### Authentication Log Findings

Record observed login events here.

### Security Recommendations

Document security improvements based on actual findings.

## Next Steps

- SSH public key authentication
- SSH hardening
- Linux security auditing
- Automated log analysis
