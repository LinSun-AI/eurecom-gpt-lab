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
50.1
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
| IaaS VM | <!--answer:vm_cold_ms-->252.4<!--/answer--> | <!--answer:vm_warm_median_ms-->212.2<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->2237.5<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->632.0<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1028.8<!--/answer--> | <!--answer:function_warm_median_ms-->132.5<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->

```text
Warm request latency histogram (20 requests per platform)
Each # represents one request. Unit: milliseconds.

VM (Compute Engine)
    0-199   |                      0
  200-399   | #################### 20
  400-599   |                      0
  600-799   |                      0

Cloud Run
    0-199   | #########            9
  200-399   |                      0
  400-599   |                      0
  600-799   | ###########          11

Cloud Function
    0-199   | ############         12
  200-399   |                      0
  400-599   |                      0
  600-799   | ########             8
```

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
Compute Engine does not automatically create additional VMs. All requests are handled by the single e2-micro instance, limited by its CPU and memory. With increasing concurrent requests, latency may increase as requests queue. Cloud Run automatically creates container instances to handle traffic, but I configured a maximum of three instances using `--max-instances=3`. Cloud Functions Gen 2 also scales automatically by creating additional managed instances. I did not explicitly set a maximum instance count for the Function, so its scaling is constrained by the platform's configuration and quotas.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
Compute Engine required the most manual work. I created a custom VPC, subnet, firewall rules, service account, and e2-micro VM. I then connected through SSH, installed Docker, authenticated with Artifact Registry, pulled the image, and started the container. VM maintenance and operating system updates remain my responsibility. Cloud Run was simpler because I could deploy the existing Docker image directly, configuring the listening port, scaling limits, and public access. Cloud Functions required the Python source code, runtime, entry point, and HTTP trigger without manually managing a container. However, I encountered Cloud Build IAM permission errors during deployment and had to configure the necessary service account permissions.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
```text
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-142795087360.us-central1.run.app
LAST DEPLOYED BY: service-142795087360@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-11-01T16:11:41.959146Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-142795087360.us-central1.run.app
LAST DEPLOYED BY: Lin.Sun@eurecom.fr
LAST DEPLOYED AT: 2026-11-01T16:06:46.045365Z
```
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The service list shows both `echo-bot` and `echo` as Cloud Run services. This indicates that my second-generation Cloud Function runs on Cloud Run infrastructure, even though I deployed it using the Cloud Functions interface. Both platforms use managed containers, support automatic scaling, and may scale down to zero when idle. Therefore, their first requests may require starting new instances, causing cold-start delays. My measurements showed 2237.5 ms for Cloud Run and 1028.8 ms for Cloud Functions. Both also exhibited two distinct clusters of warm request latency, although the medians differed. The main difference is the deployment abstraction: Cloud Run uses an explicitly built container image and exposes container configuration, while Cloud Functions accepts Python source code and an entry point, with Google managing the build and underlying container deployment.
<!--/answer-->