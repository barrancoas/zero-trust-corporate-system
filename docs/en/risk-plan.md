# Risk Management Plan

**Project:** Zero Trust Corporate System
**Author:** Asier Barranco  
**Date:** 14/04/2026  
**Version:** 1.0

---

## 1. Introduction

This document identifies the occupational and technical risks associated with the development of this project, along with the preventive measures defined to minimise their impact. It has been produced in accordance with the requirements of the training cycle and applies to all work carried out during the project period, whether in the classroom or in a cloud environment.

---

## 2. Work Environment

The project is developed in the following physical and digital contexts:

- **Classroom:** standard workstation with monitor, keyboard and mouse. Seated work for extended periods.
- **Cloud environment (AWS):** remote administration via browser and SSH. No physical infrastructure is handled directly beyond the local workstation.
- **On-premise virtualisation (VirtualBox):** virtual machines running on the local workstation. No additional physical hardware involved.

---

## 3. Risk Identification and Preventive Measures

### 3.1 Occupational and Ergonomic Risks

| # | Risk | Likelihood | Severity | Preventive Measures |
|---|------|-----------|----------|---------------------|
| R01 | Musculoskeletal disorders due to prolonged sitting | Medium | Medium | Maintain correct posture. Take a short break every 45–60 minutes. Adjust chair height and monitor distance. |
| R02 | Eye strain from prolonged screen exposure | Medium | Low | Follow the 20-20-20 rule: every 20 minutes, look at something 20 feet away for 20 seconds. Adjust screen brightness to room lighting. |
| R03 | Repetitive strain injury (wrist/hand) from keyboard and mouse use | Low | Medium | Use ergonomic positioning. Avoid resting wrists on hard surfaces while typing. |
| R04 | Fatigue and stress from project deadlines | Medium | Medium | Plan work in manageable sprint tasks. Avoid overloading single sessions. Use ProofHub to keep a clear view of progress. |

---

### 3.2 Electrical and Equipment Risks

| # | Risk | Likelihood | Severity | Preventive Measures |
|---|------|-----------|----------|---------------------|
| R05 | Electric shock from faulty classroom equipment | Low | High | Do not manipulate cables or internal hardware. Report any damaged equipment to the instructor immediately. |
| R06 | Overheating of workstation due to high VM load | Low | Medium | Monitor CPU and RAM usage. Avoid running more virtual machines simultaneously than the hardware can support. Close unused applications. |
| R07 | Accidental data loss due to hardware failure | Low | High | Commit work to GitHub regularly. Never keep the only copy of a document locally. |

---

### 3.3 Digital and Cybersecurity Risks

| # | Risk | Likelihood | Severity | Preventive Measures |
|---|------|-----------|----------|---------------------|
| R08 | Accidental exposure of AWS credentials | Medium | High | Never commit access keys or secrets to GitHub. Use environment variables or AWS IAM roles. Rotate credentials immediately if exposed. |
| R09 | Unintended costs in AWS due to misconfigured resources | Medium | High | Set a billing alert at 80% of the available budget. Review running instances daily. Terminate unused resources after each session. |
| R10 | Unintended access to production systems during Red Team exercises | Low | High | All attack simulations are performed exclusively within the isolated lab environment. No tests are run against real external systems. |
| R11 | Loss of project data due to accidental deletion | Low | High | Work on branches in GitHub. Never force-push to the main branch without review. |
| R12 | Dependency on a single cloud region causing service disruption | Low | Medium | Document all configurations so the environment can be reproduced. Keep infrastructure documentation updated. |

---

### 3.4 Organisational Risks

| # | Risk | Likelihood | Severity | Preventive Measures |
|---|------|-----------|----------|---------------------|
| R13 | Scope creep — adding features beyond the defined backlog | Medium | Medium | Strictly follow the defined backlog. Any scope change must be evaluated against available time before being added to a sprint. |
| R14 | Poor time estimation causing sprint overload | Medium | Medium | Include a time buffer in each sprint. If a task is not completed, carry it over to the next sprint and document the reason in the retrospective. |
| R15 | Insufficient documentation during implementation | Medium | High | Document each component as it is deployed, not at the end of the sprint. Follow the definition of done: no task is complete without its documentation committed. |

---

## 4. Emergency Procedures

- **Medical emergency in the classroom:** notify the instructor immediately and follow the centre's emergency protocol.
- **Electrical incident:** do not touch affected equipment. Disconnect power at the panel if safe to do so. Notify the instructor.
- **AWS credential leak:** immediately revoke the exposed credentials from the AWS IAM console, rotate all related keys, and review CloudTrail logs to assess any unauthorised activity.
- **Accidental data loss:** check GitHub commit history and branch backups before attempting any recovery.