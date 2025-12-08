# HTB Popcorn - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2025-12-8

**Target:** 10.129.14.108

**Environment:** HTB Popcorn (Retired Machine)

---

## 0. Lesson Learned

Look deeper. When you couldn't manipulate the torrent file, the answer lied in editing the png screenshot of the torrent.

---

## 1. Executive Summary

The objective of this engagement was to perform a controlled penetration test against the **HTB Popcorn** host. The goal was to identify exploitable vulnerabilities, achieve user and root compromise, and document all findings with reproducible steps.

The assessment was successful. A chain of web application vulnerabilities in the *Torrent Hoster* software allowed authenticated file upload manipulation, leading to remote code execution (RCE). Privilege escalation was achieved via a vulnerable 2.6.31 Linux kernel using the **full-nelson** exploit. Full system compromise was obtained.

---