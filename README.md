# Zero Trust Corporate System

> Design and implementation of a corporate Zero Trust ecosystem with federated identity and security auditing.

**Author:** Asier Barranco  
**Cycle:** CFGS ASIX — Cybersecurity Profile  
**Centre:** Institut Tecnològic de Barcelona  
**Academic year:** 2025–2026

---

## Context

Traditional perimeter-based security models are no longer enough. In a landscape shaped by remote work and cloud migration, the idea of a trusted internal network has become obsolete. Modern threats routinely exploit compromised credentials or vulnerable internal devices to operate from inside the network — making the perimeter itself irrelevant.

This project responds to that reality by designing and deploying a **Zero Trust architecture**. In this model, identity becomes the new perimeter. Every access request — regardless of where it comes from — must be explicitly verified, authorised and encrypted before reaching any corporate resource. Nothing is trusted by default, not even traffic coming from inside the network.

---

## What This Project Does

The project builds a **hybrid corporate infrastructure** combining on-premise resources and a public cloud environment (AWS). It hosts fully isolated corporate services — document management and team communications — behind a single, tightly controlled entry point.

The main goals are:

- Deploy a hybrid infrastructure (on-premise + AWS) that keeps corporate services completely isolated from the public internet.
- Centralise identity management through an Active Directory that governs all access control policies across the organisation.
- Enforce strong authentication — every access must pass through an identity proxy with Single Sign-On, Multi-Factor Authentication and TLS/HTTPS encryption.
- Validate the architecture through offensive and defensive security testing.

---

## Architecture

The system is built around three layers:

**Layer 1 — Identity & Control (on-premise)**
A central Active Directory server stores and manages all corporate identities, groups and access policies. It is the trust anchor of the entire system — no resource is accessible without going through it.

**Layer 2 — Services & Data (AWS private subnets)**
Corporate web applications alongside their relational databases, deployed in private subnets with no direct routing from the public internet. Network segmentation is a hard requirement, not an option.

**Layer 3 — Perimeter (AWS public subnet)**
A reverse proxy acts as the sole entry point for all incoming traffic. It integrates with the identity federation layer and enforces strict firewall policies and automated certificate management. If a request does not pass identity verification, it never reaches the internal network.

---

## Security Validation — Purple Team

The project concludes with a practical offensive/defensive validation:

- **Red Team:** Documented attack simulations — brute force against the SSO portal, credential theft via phishing, and session hijacking attempts.
- **Blue Team:** Evidence of mitigation — dynamic IP blocking, MFA repudiation of compromised credentials, log analysis and a formal incident response report.

---

## Methodology

The project follows **SCRUM** with two-week sprints. Each sprint is documented with a planning record and a review/retrospective, both committed to this repository as Markdown files with ProofHub board screenshots as evidence.

No custom software is developed. The project focuses entirely on the deployment, integration and hardening of open source solutions. All technical documentation is written in **technical English (B2 level)** with a parallel version in Spanish.