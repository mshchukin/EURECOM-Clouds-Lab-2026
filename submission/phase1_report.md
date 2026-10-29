# Phase 1 — Comparison Report

## 1. Setup facts

**Your image size**, in MB:
<!--answer:size-->
49
<!--answer:size-->

**The custom VPC and subnet you created**:
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

## 3. Per-platform analysis

The IaaS VM relies on a single instance, Cloud Run scales up to 3 instances via flags, and Cloud Functions scale automatically within platform limits. Deployment effort decreases from manual IaaS provisioning to managed container and source-based workflows on Cloud Run and Functions.

## 4. Architecture

Gen-2 Cloud Functions operate on Cloud Run infrastructure, resulting in comparable performance metrics while offering varying levels of underlying container abstraction.
