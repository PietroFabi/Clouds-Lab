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
49
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
| IaaS VM | <!--answer:vm_cold_ms-->252.9<!--/answer--> | <!--answer:vm_warm_median_ms-->211.8<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->1838.7<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->633.0<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1660.3<!--/answer--> | <!--answer:function_warm_median_ms-->636.3<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
------------------------------------------------------------
[OK  ] vm        cold=  252.9ms  warm_median=  211.8ms
            #############
[OK  ] cloudrun  cold= 1838.7ms  warm_median=  633.0ms
            ########################################
[OK  ] function  cold=  1660.3ms  warm_median=  636.3ms
            ########################################
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
The IaaS VM creates nothing automatically to absorb load; its hard limit is the single machine we provisioned, dictated by the `--machine-type=e2-micro` flag. Cloud Run automatically creates new container instances to handle concurrent requests; its hard limit is 3 instances, dictated by the `--max-instances=3` flag we passed during deployment. Cloud Functions (gen 2) also scale automatically by spinning up new instances to absorb load; its hard limit is defined by the platform's default maximum instance quota unless explicitly overridden.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
The VM (IaaS) required the most effort: we manually configured a custom VPC and firewall rules (TCP 8080 and 22), provisioned the VM, connected via SSH, installed Docker, authenticated with Artifact Registry, and ran the container. Cloud Run (Container PaaS) was much simpler: after building and pushing the image, a single command deployed it, automatically handling HTTPS and scaling. Cloud Functions (Serverless PaaS) required the least effort: we simply deployed the raw Python source code, offloading both the container build process and infrastructure management entirely to Google Cloud.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-253764727619.us-central1.run.app
LAST DEPLOYED BY: service-253764727619@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-12T20:57:02.738754Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-253764727619.us-central1.run.app
LAST DEPLOYED BY: Pietro.Fabi@eurecom.fr
LAST DEPLOYED AT: 2026-10-12T20:51:04.792831Z
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The output reveals that Google Cloud Functions (gen 2) are actually deployed as Cloud Run services under the hood. Because both the Cloud Run deployment and the Cloud Function run on the exact same underlying serverless infrastructure, their performance characteristics and cold start latencies are nearly identical. 

The genuine difference between the two lies purely in the developer experience and deployment path. For Cloud Run, I had to manually write a Dockerfile, build the container image, and push it to Artifact Registry. For the Cloud Function, I only provided the raw Python source code, and Google's internal build tools (as seen by the gcf-admin-robot deployer) automatically handled the containerization and deployment process for me.
<!--/answer-->
