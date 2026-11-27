# votesecure-k8s

A learning project: the VoteSecure voting app (React frontend, FastAPI backend, PostgreSQL) on a local
Kubernetes cluster (kind), with a Helm chart, a security scan in CI, autoscaling, GitOps with Flux and
failure drills. Everything was built and measured on one laptop (2 CPUs, about 3.9 GB RAM). It is a lab,
not a production setup. The limits at the end are part of the result.

The app source and the image builds are in another repo: Phvl-0/votesecure-fullstack.

## What runs

    GitHub repo (main is protected) <--- pull request + Checkov check
            |
            |  read-only deploy key, polled every 2 minutes
            v
    namespace flux-system:  source-controller --> helm-controller
                                                       |  helm upgrade (release "votesecure")
                                                       v
    namespace votesecure (Pod Security: restricted)
      ingress-nginx --> web (nginx, 8080) --> api (FastAPI, 8000, 1 to 3 pods) --> postgres (5432, volume 1Gi)

    NetworkPolicies (ingress only): web accepts only the ingress controller, api only web, postgres only api.
    kube-system: metrics-server feeds the autoscaler (HPA) on the api.

| Part | What it is |
|---|---|
| web | nginx serving the React build, container port 8080, Service port 80 |
| api | FastAPI on 8000, logins hash passwords with bcrypt, HPA 1 to 3 pods at 50% CPU of the 100m request |
| postgres | postgres:16-alpine as uid 70, one pod, volume postgres-data (1Gi), Service name db |

The api and web images are pulled from ghcr.io by full commit SHA (imageTag in values.yaml).

## Repository layout

| Path | Purpose |
|---|---|
| helm/votesecure/ | the chart: web, api, postgres, Services, Ingress, volume claim, NetworkPolicies, HPA |
| flux/ | Flux GitRepository and HelmRelease, applied by hand with kubectl |
| flux-install-min.yaml | Flux v2.10.0, only source-controller and helm-controller, generated with flux install --export |
| metrics-server-kind.yaml | metrics-server v0.9.0 with one added arg, --kubelet-insecure-tls (header says why) |
| ingress-nginx-kind.yaml | ingress-nginx v1.15.1 (archived upstream, lab use only) |
| namespace.yaml | namespace votesecure with the Pod Security labels, applied by kubectl, not by Helm |
| .github/workflows/k8s-scan.yml | Checkov scan of the rendered chart on every pull request |
| api.yaml, web.yaml, postgres.yaml, pvc.yaml, ingress.yaml, api-service.yaml, db-service.yaml, web-service.yaml | the first hand-written manifests. Superseded by the chart. Do not apply. |

The history contains one failure drill: a bad image tag was merged and then reverted (the two commits whose
messages start with DRILL and Revert DRILL).

## How a change reaches the cluster

1. Work on a branch and open a pull request. Branch protection on main refuses direct pushes (proven: GH006)
   and requires the check named checkov. Rules: enforce_admins on, no force push, no deletion, no required reviews.
2. CI renders the chart with helm template and scans it with Checkov (kubernetes framework).
3. After the merge, source-controller fetches main (every 2 minutes) and helm-controller runs the upgrade of the
   release votesecure (every 2 minutes). Drift detection is on, so manual changes to chart objects are reverted.

## What was proven, with numbers

Each item was measured on this cluster. "Proven" means a check from both sides where that was possible.

**Pod hardening and CI**
- Checkov found 30 issues at first: 26 fixed and each checked in the kernel (for example Seccomp: 2 in
  /proc/1/status, the service account token folder gone), 4 accepted with a written reason in the workflow.
- Gate proof: green run 45022872717, red on a planted flaw (CKV_K8S_13) 45025322309, green after the revert 45034018550.
- At C3 the scan read Passed 262, Failed 0. Later runs were green, but I did not read the count again.
- Namespace enforces the Pod Security level restricted. A plain busybox pod was refused, and the message named
  allowPrivilegeEscalation, capabilities, runAsNonRoot and seccompProfile.

**NetworkPolicies**
- A pod that is not the api reached postgres with no policy ("accepting connections"), was blocked with the
  policy ("no response"), and reached it again after the policy was deleted (control test).
- Enforcement was also shown for web and api with one test each, without a before-and-after.

**Autoscaling** (metrics-server v0.9.0, HPA 1 to 3 pods, 50% CPU of the 100m request)
- The upstream metrics-server manifest stayed 0/1: its log said the kubelet certificate has no IP SANs. With
  --kubelet-insecure-tls it became ready and kubectl top worked.
- Load test, run 2: 4 workers, 240 s, wrong-password logins for an existing account (so bcrypt runs), sent
  straight to the api. Preflight: 401, and 20 logins took 14 s.
- From the Kubernetes events: scale-up from 1 to 3 pods 12 s after the load started, scale-down to 1 pod
  5 min 26 s after the load ended. Peak CPU sample 397% of the request (495% in run 1). Lowest available RAM
  2422 MB. No restarts during the test.
- A helm upgrade while the api had 3 pods left it at 3: the chart omits the api replicas when the HPA is on.
- Observed twice: browser logins alone made the HPA scale the api to 3. One bcrypt login is enough at a 100m request.
- This shows the mechanism, not capacity. No requests per second or latency was measured.

**GitOps with Flux v2.10.0**
- Flux took over the existing release (Helm revision 12 on takeover). The same pods, the same user count and
  both Secrets were unchanged.
- A merge to main changed the cluster with nothing run by hand: 3 min 54 s in one run, about 1 s in another
  (two polling timers of 2 minutes each, so 0 to about 4 minutes).
- Web was scaled to 2 by hand: reverted within about 60 s (helm-controller log: correct cluster drift).

**Failure drills**

| Drill | Detection | Recovery | Data lost |
|---|---|---|---|
| Delete the api pod | replacement seen in 2 s or less | ready in 17 s | 0 (users 4 to 4) |
| Bad image tag merged via pull request | first pull error at 19 s, HelmRelease not ready after 5 min 9 s (our 5 minute timeout). Old pods served the whole time. | 138 s after the revert merge | 0 |
| Delete the database volume (Flux suspended first, resumed after) | login errors within seconds | login back after about 28 s: Flux recreated the volume 7 s after resume, restore from a manual dump took 1 s | 0, but only because nothing was written after the manual dump |

- The restore was checked by comparing row counts of every table (identical) and by a browser login. Only the
  users table has rows (4), so restoring tables with data was not tested.
- Nothing alerts a person in any drill. Detection means the failure was visible to a probe or to kubectl.

**Web restarts.** Web was restarted by the kubelet (exit code 0) when its liveness probe, with a 1 s timeout,
timed out while the node was stalled (the kubelet log also showed slow etcd and API server checks). The timeout
is now 5 s and the new pod had 0 restarts after 23 hours. That is one pod and one day: not proof.

## Rebuild

Written from what was done, not re-run end to end on a clean machine. The exact kind create command was not
recorded; the cluster has one node (2 CPUs, kindest/node v1.37.0).

Tools used: Docker Engine, kind v0.33.0, kubectl v1.37.2, Helm v4.3.1.

Made by hand, never in Git: the two Secrets, the deploy key, the branch protection rules on GitHub.

1. Create the cluster:

        kind create cluster --name votesecure --image kindest/node:v1.37.0

2. Namespace first (it carries the Pod Security labels), then the ingress controller and metrics-server:

        kubectl apply -f namespace.yaml
        kubectl apply -f ingress-nginx-kind.yaml
        kubectl apply -f metrics-server-kind.yaml

3. Secrets in the namespace votesecure (type the values, they are not echoed and not saved in history):

        read -rs PGPASS; read -rs SKEY
        kubectl create secret generic votesecure-secrets -n votesecure \
          --from-literal=POSTGRES_PASSWORD="$PGPASS" --from-literal=SECRET_KEY="$SKEY"
        unset PGPASS SKEY

   The api and web images come from ghcr.io. Either create a read-only package token and a pull secret named
   ghcr-pull (kubectl create secret docker-registry ghcr-pull ...), or load the images with kind load docker-image.
   The token used here has expired.

4. Flux. Create a key pair, add the public key to this repo as a read-only deploy key, check GitHub's host key
   fingerprint against the one in GitHub's documentation, then:

        kubectl apply --server-side -f flux-install-min.yaml
        kubectl create secret generic flux-repo-key -n flux-system \
          --from-file=identity=KEYFILE --from-file=identity.pub=KEYFILE.pub --from-file=known_hosts=KNOWN_HOSTS_FILE
        kubectl apply -f flux/git-source.yaml
        kubectl apply -f flux/helmrelease.yaml

   Without Flux, install the chart directly: helm install votesecure helm/votesecure --namespace votesecure

5. The database starts empty on a fresh install (the chart resets it). Restore a dump with psql if you have one.
6. Open the app: kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8091:80 and visit
   http://localhost:8091
7. GitHub settings: protect main (required check checkov, enforce_admins, no force push, no deletion).

## Honest limits

- A green scan only means nothing was found outside the written skip list. Four findings are accepted: image
  digest (CKV_K8S_43), pull policy Always (CKV_K8S_15), secrets as environment variables (CKV_K8S_35), high UID (CKV_K8S_40).
- The root manifests (metrics-server, Flux install, flux/, ingress) are not scanned by Checkov. Only the chart is.
- Images are pinned by tag, not by digest: postgres, metrics-server, ingress-nginx and the Flux controllers.
  The api and web images use full commit SHA tags.
- The ghcr pull token expired. A new image cannot be pulled until there is a new token or the image is loaded
  with kind load. The bad-tag drill could not tell a missing tag from the expired token.
- Postgres hardening was tested on kind's local volume only. NetworkPolicies are ingress only (no egress rules)
  and enforcement was proven on this kind cluster only. The Pod Security audit label is set, but audit logging was not checked.
- The api has no database retry (it exits if the database or DNS is not ready at start, and Kubernetes restarts
  it). Its probe is tcpSocket only: it proves the port opens, not that logins work.
- Backups are manual (pg_dump). The real data loss in a volume loss is everything since the last dump.
- Only one node with 2 CPUs. The control plane stalls and restarts sometimes (the scheduler and controller-manager
  have restarted hundreds of times, cause not found), and a laptop restart restarts every pod.
- metrics-server skips kubelet certificate verification, and its APIService skips certificate verification
  (upstream default). Fine for a one-node lab only.
- ingress-nginx is archived upstream and is reached only by port-forward. The manifest in this repo records
  no source or checksum.
- Flux: helm-controller holds cluster-admin (the Flux default). kustomize-controller is not installed, so the
  files in flux/ are applied by hand. There are no alerts (no notification-controller). The deploy key has no
  passphrase and exists in ~/.ssh and in a cluster Secret. Deleting the HelmRelease runs helm uninstall and would
  remove the app and the postgres volume. helm-controller logged one status-conflict error right after a drift
  correction; the cluster state was correct and the cause was not found.
- The chart's database is reset on each fresh install. The Secrets are deliberately not in the chart. Only one
  install per namespace works (fixed object names). The first hand-written manifests in the repo root duplicate the chart.
- Not added: a Trivy scan, a scheduled backup, a rebuild test from scratch.
