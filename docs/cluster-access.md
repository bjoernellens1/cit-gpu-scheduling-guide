# Getting started: cluster access

Before you can submit any of the jobs in this guide, you need a working
`kubectl` context pointed at the cluster, with RBAC scoped to your own
namespace/project (not cluster-admin).

This guide assumes a **Rancher-managed** cluster (a common pattern for
multi-tenant GPU clusters — Rancher handles SSO/OIDC login and issues
scoped kubeconfigs per user). If your cluster is managed differently, the
concepts (get a kubeconfig, verify your RBAC scope, know your token's
expiry) still apply — just substitute your own cluster's provisioning
process.

## 1. Get your kubeconfig

1. Log in to your cluster's Rancher UI via your organization's SSO
   (Rancher supports OIDC/SAML/LDAP/GitHub/Azure AD and others — use
   whichever your admin has configured).
2. Navigate to the cluster you'll be submitting jobs to.
3. Click **Download KubeConfig** (usually top-right, or under **Cluster
   Dashboard**). This downloads a `kubeconfig` file scoped to your Rancher
   user identity, not a shared/admin credential.
4. Set it as your active config, or merge it into `~/.kube/config`:

   ```bash
   export KUBECONFIG=/path/to/downloaded/kubeconfig
   kubectl get nodes   # sanity check — should list cluster nodes
   ```

**Rancher-issued kubeconfigs typically embed a bearer token, not a
long-lived client certificate.** That token has an expiry (commonly a few
hours to a day, depending on your Rancher admin's session-timeout
policy). If commands that used to work suddenly fail with `Unauthorized`
or `the server has asked for the client to provide credentials`, the fix
is almost always: **re-download the kubeconfig from the Rancher UI** — the
token has expired, not the cluster. This is expected behavior for
session-scoped tokens, not a bug.

## 2. Verify your RBAC scope before submitting anything

A Rancher-issued kubeconfig is scoped to whatever project/namespace(s)
your account has been granted access to — not cluster-wide. Confirm what
you can actually do before assuming a command failure is a cluster problem
rather than a permissions boundary:

```bash
# What can I do in my own namespace?
kubectl auth can-i --list -n <your-namespace>

# Can I create Jobs/PodGroups here specifically?
kubectl auth can-i create jobs.batch -n <your-namespace>
kubectl auth can-i create podgroups.scheduling.run.ai -n <your-namespace>

# What namespaces can I see at all?
kubectl get namespaces
```

If `create podgroups.scheduling.run.ai` comes back `no` but you need gang
scheduling (see [gang-scheduling.md](gang-scheduling.md)), that's a real
RBAC gap to raise with your cluster admin — not something to work around
by requesting a broader kubeconfig.

## 3. Know your queue and priority class before submitting

Ask your cluster admin (or check with `kubectl get queue -A` if your RBAC
allows it) which **KAI queue** and **PriorityClass** your account/team is
expected to use — see [priority-queues.md](priority-queues.md). Submitting
into the wrong queue, or with a priority class you're not supposed to use,
is a common first-time mistake that either gets your job stuck Pending
(insufficient quota) or, worse, preempts someone else's legitimate work.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Unauthorized` / `the server has asked for the client to provide credentials` | Rancher session token expired | Re-download kubeconfig from Rancher UI |
| `Error from server (Forbidden): ... is forbidden: User "..." cannot ...` | RBAC scope doesn't cover that action/namespace | Check `kubectl auth can-i`, raise with cluster admin if you believe you should have access |
| Pod stuck `Pending` with no scheduling errors in `kubectl describe pod` | Wrong/insufficient queue quota | Check `kubectl get queue -n kai-scheduler <your-queue> -o yaml` for the `resources.gpu.quota`/`limit` your queue actually has |
| `kubectl` commands hang or time out entirely | Network path to the cluster API server is down (not a kubeconfig problem) | Confirm with a basic `ping`/`curl` to the API server host; this is an infra issue, not an RBAC one |
