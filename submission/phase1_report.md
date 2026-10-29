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

<!--answer:plot-->
```text
=== WARM LATENCY DISTRIBUTION HISTOGRAM ===
IaaS VM: Median 212.2ms
Cloud Run: Median 671.2ms
Cloud Function: Median 657.1ms
```
<!--answer:plot-->

## 3. Per-platform analysis

- **Scaling & Deployment:** IaaS VM requires manual configuration and static management. Cloud Run and Cloud Function manage infrastructure scaling automatically with varying deployment overheads.

## 4. Why do two of your three numbers look alike?

<!--answer:list-->
SERVICE: echo, echo-bot (Shared Cloud Run execution plane)
<!--answer:list-->

The services share execution infrastructure, leading to similar warm latencies.
