# Phase 1 — Comparison Report

## 1. Setup facts

**Your image size**, in MB, from gcloud artifacts docker images list (number only, e.g. 142 ):
<!--answer:size-->
49
<!--answer:size-->

**The custom VPC and subnet you created** (names only):
<!--answer:vpc-->
echo-net
<!--answer:vpc-->

<!--answer:subnet-->
echo-subnet
<!--answer:subnet-->

## 2. Measurements

Platform | Cold (ms) | Warm median (ms)
---|---|---
IaaS VM | <!--answer:vm_c-->241.7<!--answer:vm_c--> | <!--answer:vm_w-->212.2<!--answer:vm_w-->
Cloud Run | <!--answer:cr_c-->694.7<!--answer:cr_c--> | <!--answer:cr_w-->671.2<!--answer:cr_w-->
Cloud Function | <!--answer:cf_c-->1913.7<!--answer:cf_c--> | <!--answer:cf_w-->657.1<!--answer:cf_w-->

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart measure.py prints). This is read by a human, not the grader:
<!--answer:plot-->
```text
=== WARM LATENCY DISTRIBUTION HISTOGRAM ===
IaaS VM        [212.2ms] : #############
Cloud Run      [671.2ms] : ########################################
Cloud Function [657.1ms] : #======================================
```
<!--answer:plot-->

## 3. Per-platform analysis

**Scaling behaviour.** Imagine requests arriving one at a time, then ten at once, then a hundred. For each of the three platforms, answer concretely:
<!--answer:scaling-->
The IaaS VM creates nothing automatically under load; it handles concurrent spikes sequentially or via application threading on a single persistent guest OS execution context. Its hard ceiling stems from the hardware constraints of the selected e2-micro virtual machine (1 vCPU, 1 GB RAM). Cloud Run dynamically instantiates horizontal container instances up to the strict limit of 3 instances specified by the --max-instances flag. The gen-2 Cloud Function dynamically spawns independent stateless container worker runtimes behind an infrastructure-managed autoscaling engine, up to regional platform concurrency limitations.
<!--answer:scaling-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the three running? (~100 words)
<!--answer:effort-->
For the IaaS VM path, deployment required building a multi-stage Dockerfile, provisioning a custom VPC network, subnets, and manual ingress firewall rules (TCP 22/8080). Operating system dependencies, manual package retrieval, and explicit daemon authentication had to be explicitly executed inside the instance via secure shell. For Cloud Run, deployment required building and registering a standard pre-compiled image to the Artifact Registry, followed by a managed container declarative engine request. For the Cloud Function path, the underlying deployment system automated artifact generation directly from the local raw source directory code layout without manual compilation.
<!--answer:effort-->

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run gcloud run services list and paste the **complete output** here, unedited:
<!--answer:list-->
✔
SERVICE: echo
REGION: us-central1
URL: https://run.app
LAST DEPLOYED BY: service-161110259588@://gserviceaccount.com
LAST DEPLOYED AT: 2026-10-29T22:41:12.285265Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://run.app
LAST DEPLOYED BY: Bloorger@gmail.com
LAST DEPLOYED AT: 2026-10-29T22:10:28.254762Z
<!--answer:list-->

Now explain what that output tells you about how your Function is actually being run, why its cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between the two deployment paths. (~120 words)
<!--answer:architecture-->
The gcloud output explicitly reveals that gen-2 Cloud Functions are executed as standardized services running on top of Google Cloud's serverless container layer. The function `echo` appears directly as an isolated native deployment endpoint alongside the standard containerized app instance `echo-bot`. Because both leverage the exact same execution back-plane, their warm operational latencies are nearly identical (~657ms and ~671ms), using the same regional routing systems. The substantial difference emerges during a cold boot sequence (1913.7ms versus 694.7ms); Cloud Run consumes a pre-compiled image fetched directly from Artifact Registry, whereas the Cloud Function engine handles an on-demand container assembly and setup phase directly from source code variables.
<!--answer:architecture-->
