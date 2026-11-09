---
name: ultraviolet-automatic-fixes
description: Let Ultraviolet Observability runbooks fix incidents on their own, safely — start from shadow mode's evidence, allow specific actions in specific namespaces per site, check the autofix service account's Kubernetes permissions, tune guardrails, and know the kill switches. Use when someone wants runbooks to act automatically, auto-remediation, self-healing, or to scope or turn off automatic fixes.
---

# Set up automatic fixes

A runbook at `autonomy: automatic` runs its fix with nobody approving, when an
alert's or SLO's investigation matches and no guardrail holds it. It acts at a
site only if **that site's chart allows it**, as a dedicated service account
(`<release>-autofix`) with exactly the rights the allowed actions need. Then it
checks the fix worked, and says so on the alert's channels. Reference:
https://docs.uvobservability.com/guides/runbooks/#automatic-fixes

Prerequisites: actions are on (`queryApi.actions.enabled`, and
`remoteSites: true` for sites), and each site to act at runs the actuator
(https://docs.uvobservability.com/guides/actions/#adding-the-actuator-to-a-site).

## 1. Start from evidence, not hope

Recommend **shadow mode first** for each runbook (`autonomy: shadow`): it
proposes fixes for people to approve, and records what it would have done on
its own. On **Monitoring → Runbooks**, its 30-day record shows how often it
would have acted, whether people ran the same fix, whether that worked, and how
much sooner it would have been. Move a runbook to automatic only when that
record agrees with what the team did.

## 2. Allow it at the site

In the **site's** chart values (`ultrvlt-site`), list only what's needed:

```yaml
actuator:
  enabled: true
  commandPublicKey: <from the core's Sites page>
  mode: passthrough            # how people's own actions run; unrelated to automatic
  automatic:
    enabled: true
    actions: [rollback]        # restart, scale, rollback, delete_pod, cordon, uncordon, drain
    namespaces: [shop]         # ["*"] for all; node actions need only the action
```

Pick actions per runbook: `rollback` for bad deploys, `scale` to restore a
scaled-down workload, `delete_pod` for one bad pod, `drain`/`cordon` for node
pressure (these grant a ClusterRole on nodes, so allow them deliberately).

On the **core's own cluster**, set the same under
`queryApi.actions.actuator.automatic` in the core chart, unless the core's
gateway requires enrolled client certificates; then set it on the co-located
connector's site chart instead.

**Managing RBAC yourselves** (GitOps, policy engines): set
`automatic.serviceAccount.create: false` with its `name`, and
`automatic.rbac.create: false`. The chart then grants only the actuator's right
to impersonate that one account.

Preview, ask, and upgrade the site chart with `--reset-then-reuse-values`.

## 3. Check the account's rights

Show the user that the account can do exactly what was allowed and no more
(the account is `<release>-autofix` in the release's namespace, so
`ultrvlt-site-autofix` for a release named `ultrvlt-site`):

```bash
SA=system:serviceaccount:ultrvlt:ultrvlt-site-autofix
kubectl auth can-i patch deployments -n shop --as "$SA"          # yes, for rollback in shop
kubectl auth can-i patch deployments -n kube-system --as "$SA"   # no
kubectl auth can-i delete pods -n shop --as "$SA"                # no, unless delete_pod allowed
kubectl auth can-i get secrets -n shop --as "$SA"                # no
```

Then, on the core's **Sites** page, the site's actuator should be connected
and list the allowed actions and namespaces.

## 4. Turn the runbook on

In the portal (**Monitoring → Runbooks**, editor) or in the core chart:

```yaml
queryApi:
  runbooks:
    - slug: rollback-bad-deploy
      enabled: true
      autonomy: automatic
      cooldown: 15m        # least time between actions on one target
      max_per_hour: 2      # most actions an hour on one target (up to 12)
```

Built-in slugs: `rollback-bad-deploy`, `restore-scaled-down`,
`delete-bad-pod`, `drain-node-under-pressure`. Scope a runbook with
`scope: {namespaces: [...]}` as well, if it should act more narrowly than the
site allows.

## 5. Guardrails it obeys

It proposes for a person instead (and says why) when:

- automatic fixes are switched off (the switch on the Runbooks page), or it's
  inside a **freeze window** set there (release windows, peak trading);
- the same fix **didn't work** on that target in the last hour;
- anyone acted on the target within the **cooldown**, or it hit
  **max_per_hour**;
- **blast radius**: a pod with fewer than two siblings serving, or a node
  action while another node is cordoned or fewer than two could take its pods;
- the site doesn't allow that action or namespace, or a person is already
  deciding on the same fix.

An investigation a person started never acts by itself.

## 6. What the user will see

- **Fixes** and the investigation show the fix as **Ran on its own**; the
  alert's channels get a notice saying it was fixed automatically.
- The fix is checked against the service's baseline (10 minutes by default).
  If it didn't work, the channels hear it and the runbook won't act on that
  target again for an hour.
- The cluster's audit log shows the change by the actuator, impersonating the
  autofix account (on GKE: `authenticationInfo.authoritySelector`).

## 7. Kill switches

Make sure the user knows all three:

- **Everywhere, now:** the automatic fixes switch on the **Runbooks** page (admin).
- **Everywhere, from the deployment:** `queryApi.actions.automatic: false` in
  the core chart. Automatic runbooks then only propose.
- **At one site, independent of the core:** `actuator.automatic.enabled:
  false` in its chart, or delete the autofix account's RoleBinding:

  ```bash
  kubectl get rolebindings,clusterrolebindings -A | grep autofix
  ```

## GitOps caution

If Argo CD or Flux manages the workload, its next sync puts the old state
back, undoing a rollback or scale. Automatic fixes don't pause sync. Scope
runbooks (and the site's `namespaces`) to workloads Git doesn't own, or make
sure the team fixes Git too after each automatic fix.
