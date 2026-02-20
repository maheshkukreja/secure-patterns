# Verification checklist

### Identity binding
- [ ] Each workload has a dedicated ServiceAccount (no reuse of the `default` SA)
- [ ] Cloud trust policies include both `namespace` and `serviceaccount` conditions (not just SA name)
- [ ] Creating a ServiceAccount with the same name in a different namespace does not grant access to another workload's cloud role

### Token configuration
- [ ] Projected token `audience` is set to the exact STS endpoint string (e.g., `sts.amazonaws.com`, not blank or `*`)
- [ ] Projected token `expirationSeconds` is set explicitly and is less than or equal to 3600s
- [ ] Pods do not mount the default ServiceAccount token (`automountServiceAccountToken: false`) when only the projected token is needed

### IAM policy scoping
- [ ] IAM Access Analyzer (or equivalent) reports no unused permissions for the workload's role
- [ ] No trust policy uses wildcard conditions on namespace, ServiceAccount, or audience claims
- [ ] IAM roles are not shared across workloads with different permission requirements

### Credential lifecycle
- [ ] The application SDK refreshes credentials before the projected token expires (observable in logs or metrics)
- [ ] Changing a pod's ServiceAccount and restarting causes the old cloud role to become inaccessible within one token TTL
- [ ] Deleting the trust policy immediately blocks new token exchanges (existing sessions drain within max session duration)

### Detection
- [ ] Cloud audit logs capture every AssumeRoleWithWebIdentity (or equivalent) call with the source ServiceAccount identity
- [ ] Alerts fire on token exchange attempts from unexpected namespaces or ServiceAccount names
- [ ] A query of all cloud trust policies returns only known, documented OIDC issuer URLs
- [ ] STS authentication failure rate is monitored, and alerts fire on spikes correlated with cluster operations
