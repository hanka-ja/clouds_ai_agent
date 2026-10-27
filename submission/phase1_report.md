# Phase 1 — Comparison Report

Copy this file to `submission/phase1_report.md`, fill in every answer slot, and commit it:

```bash
cp phase-1-echo-bot/report_template.md submission/phase1_report.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers must **match `submission/phase1_report.json`** exactly — that file is produced by
`measure.py` and is what your figures are checked against. Copy them across; do not retype from
memory.

---

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->
50
<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

---

## 2. Measurements

Copy these from `submission/phase1_report.json`. Cold is `cold_ms`; warm median is
`stats.median`. Numbers only, in milliseconds (e.g. `812.4`).

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->249.3<!--/answer--> | <!--answer:vm_warm_median_ms-->210.5<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->1739.4<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->136.6<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1098.6<!--/answer--> | <!--answer:function_warm_median_ms-->139.4<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
Phase 1 — deployment measurements
------------------------------------------------------------
[OK  ] vm        cold=  249.3ms  warm_median=  210.5ms
            ########################################
[OK  ] cloudrun  cold= 1739.4ms  warm_median=  136.6ms
            ##########################
[OK  ] function  cold= 1098.6ms  warm_median=  139.4ms
            ##########################
------------------------------------------------------------
<!--/answer-->

---

## 3. Per-platform analysis

**Scaling behaviour.** Imagine requests arriving one at a time, then ten at once, then a
hundred. For each of the three platforms, answer concretely:

- Does anything get created automatically to absorb the load — and if so, *what* exactly?
  (One platform here creates nothing at all.)
- What is the hard limit, and where does it come from — a flag you passed yourself, or the
  size of the machine you picked?

(~80 words. Your `--machine-type=e2-micro` and `--max-instances=3` are both concrete answers
to the second question.)

<!--answer:scaling_behaviour-->
For the IaaS VM, nothing is created automatically under load. Concurrency is limited by the 1 worker and 8 threads we configured in our Dockerfile CMD, and when traffic spikes, requests queue up; its hard limit is strictly determined by the `--machine-type=e2-micro` hardware capacity.
For Cloud Run and Cloud Functions, container instances are automatically created on demand to absorb load, scaling from 0 to N. Cloud Run's hard limit is enforced by the `--max-instances=3` flag we set during deployment, while Cloud Functions scales dynamically using GCP's default instance limits.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
For the IaaS VM, deployment effort was high: I built a custom VPC, subnet, and firewall rules (ports 8080 and 22), provisioned an e2-micro instance, configured IAM permissions, SSHed in, installed Docker, and manually ran the container. 
For Cloud Run, effort was moderate: I wrote a multi-stage Dockerfile, built and pushed the container image to Artifact Registry, and deployed it specifying the region, port 8080, and max-instances=3. 
For Cloud Functions, effort was minimal: I deployed raw source code directly using gcloud functions deploy; GCP automatically handled container creation via Buildpacks, networking, and runtime execution without requiring a Dockerfile or manual infrastructure management.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-191516786607.us-central1.run.app
LAST DEPLOYED BY: service-191516786607@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-27T13:02:31.515505Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-191516786607.us-central1.run.app
LAST DEPLOYED BY: hanka.jankovicova@eurecom.fr
LAST DEPLOYED AT: 2026-10-27T12:34:29.026914Z
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The output of `gcloud run services list` reveals that our `echo` Cloud Function is registered directly as a Cloud Run service, automatically created by the `gcf-admin-robot` service account. This confirms that 2nd-generation Cloud Functions are built on top of Cloud Run infrastructure. 
Because both deployment paths execute on the exact same Knative container runtime, their warm latency (~136–139 ms) and cold-start characteristics are virtually identical. 
The genuine difference lies in developer control versus automation. Cloud Run requires us to manage a multi-stage Dockerfile and push container images to Artifact Registry, offering fine-grained configuration. Conversely, Cloud Functions abstracts infrastructure entirely by using Cloud Buildpacks to compile raw source code into a container automatically.
<!--/answer-->
