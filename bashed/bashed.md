# HTB Bashed - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2025-11-24

**Target:** 10.129.22.73

**Environment:** HTB Bashed (Retired Machine)

---

## 1. Executive Summary

The assessment targeted the "HTB Bashed" host within a controlled lab environment. A full compromise was achieved, including user-level access and privilege escalation to root. The primary weaknesses were:

- Publicly accessible development directory exposing **phpbash**, granting an initial web shell. 
- Misconfigured permissions on `/scripts/` allowing a non-privileged user to modify a Python script executed by **root** via a hidden cron job.

These weaknesses enabled full system takeover.

---