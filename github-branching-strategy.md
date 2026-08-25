# **Git Branching Strategy** 

## **1. Purpose**

This document defines the standard Git branching models for all services in this product line (Angular frontend, Java microservices). It governs how code moves from development through UAT to demo/prod, and how security (VAPT) and code-quality (SonarQube) remediation work is tracked.

## **2. Branch Types**

### **2.1 Environment / Release branches (long-lived, protected)**

| **Branch pattern** | **Environment** | **Example**        |
|--------------------|-----------------|--------------------|
| release/dev/x.y.z  | DEV             | release/dev/1.0.0  |
| release/uat/x.y.z  | UAT             | release/uat/1.0.0  |
| release/demo/x.y.z | PROD (Demo)     | release/demo/1.0.0 |

- **Protected**: no direct pushes. All changes land via Pull Request only.

- Required PR checks before merge:

  - Build success

  - SonarQube quality gate pass

  - Grype VAPT scan (no new Critical/High CVEs unresolved)

- Version bump (x.y.z) happens **once per promotion cycle**, not per commit. A new release/dev/1.1.0 is cut only when the previous version has been promoted through release/demo/x.y.z (or intentionally abandoned).

### **2.2 Work branches (short-lived, unprotected)**

| **Branch pattern** | **Branched from** | **Merges into** | **Purpose** |
|----|----|----|----|
| feature/\<ticket-or-name\> | release/dev/x.y.z | release/dev/x.y.z | New features /enhancements |
| fix/sonar-\<module\> | release/dev/x.y.z | release/dev/x.y.z | SonarQube issue remediation |
| fix/vapt-\<service\> | release/dev/x.y.z (or current active env if urgent) | Same env it was cut from, then promoted forward | Grype/VAPT CVE remediation |
| hotfix/\<issue\> | release/demo/x.y.z | release/demo/x.y.z, then back-merged to uat and dev | Emergency prod-only fix |

**Naming examples:**

feature/ui-notif-panel  
fix/sonar-hub-service  
fix/vapt-nodejs-bouncycastle-cve-2024-30171  
fix/vapt-angular-nginx-alpine-baseimage  
hotfix/prod-login-500-error

## **3. Promotion Flow**

Code moves **strictly one direction**, version-locked. There are two flows: the normal forward promotion, and the hotfix exception that patches prod directly and then back-merges.

### **3.1 Forward promotion (normal flow)**

```html
   Work Branches                                                                       Next Cycle
 ┌───────────────────┐                                                          ┌───────────────────────┐
 │   feature/*       │                                                          │  Cut release/dev/     │
 │   fix/sonar-*     │                                                          │   x.y.(z+1)           │
 │   fix/vapt-*      │                                                          └────────────▲──────────┘
 └──────────┬────────┘                                                                       │
            │  PR + merge                                                                    │ after tagging
            ▼                                                                                │
 ┌───────────────────────┐  PR + merge   ┌───────────────────────┐  PR + merge   ┌───────────────────────┐
 │ release/dev/x.y.z     │ ───────────> │ release/uat/x.y.z       │ ───────────> │ release/demo/x.y.z    │
 │  (DEV pipeline)       │              │  (UAT pipeline)         │              │  (PROD/Demo pipeline) │
 └───────────────────────┘               └───────────────────────┘               └────────────┬──────────┘
       tag: dev-x.y.z                          tag: uat-x.y.z                                 │
                                                                                              ▼
                                                                                        tag: demo-x.y.z
```

Same version number (x.y.z) carries through dev → uat → demo. Once `release/demo/x.y.z` is tagged, the next cycle's `release/dev/x.y.z+1.0` is cut.

### **3.2 Hotfix flow (prod-only emergency exception)**

```html
 ┌───────────────────────┐
 │ release/demo/x.y.z    │
 └────────────┬──────────┘
              │ branch
              ▼
 ┌───────────────────────┐
 │   hotfix/<issue>      │
 └────────────┬──────────┘
              │ PR + merge back into demo
              ▼
 ┌───────────────────────┐
 │ release/demo/x.y.z    │──┐
 └───────────────────────┘  │ back-merge (same fix, both directions)
                              ├─────────────> release/uat/x.y.z
                              └─────────────> release/dev/x.y.z
```

`fix/vapt-*` follows this same hotfix path only when the CVE is Critical/High **and actively exploitable in demo/prod**; otherwise it stays in the normal forward-promotion flow (§4.2).

Rules:

- Never branch new work directly off release/uat/\* or release/demo/\*.

- Promotion dev → uat → demo is done via PR merge (or merge-train), carrying the **same version number** forward: release/dev/1.0.0 → release/uat/1.0.0 → release/demo/1.0.0.

- Once release/demo/1.0.0 is live, tag it (see §5) and cut release/dev/1.1.0 for the next cycle.

## **4. VAPT & Sonar Fix Workflow**

### **4.1 SonarQube fixes (fix/sonar-\*)**

1.  Branch off current active release/dev/x.y.z.

2.  Open PR → triggers scan-only validation pipeline (no deploy).

3.  Merge to release/dev/x.y.z, then **delete the branch immediately** to avoid stale branches cluttering the Sonar dashboard.

### **4.2 VAPT fixes (fix/vapt-\*)**

1.  Branch per-service, per-CVE (or grouped CVE batch) off release/dev/x.y.z.

2.  Remediation applied (OS package upgrade, base image bump, dependency upgrade, Docker config hardening).

3.  PR triggers Grype scan validation pipeline; result appended to the cumulative HTML VAPT dashboard.

4.  Merge → promote through dev → uat → demo normally.

5.  **Critical/High CVE actively exploitable in demo/prod**: treat as hotfix/vapt-\<...\> off release/demo/x.y.z, then back-merge to uat and dev.

## **5. Tagging**

Tag every merge into a release/\* branch:

release/dev/1.0.0 → tag: dev-1.0.0  
release/uat/1.0.0 → tag: uat-1.0.0  
release/demo/1.0.0 → tag: demo-1.0.0

Additionally, tag VAPT scan baselines independently of release versions so the dashboard can reference a known-good state:

vapt-baseline-2026-07-23

## **6. Branch Protection Rules (Jenkins / Git host)**

| **Branch** | **Direct push** | **Required PR reviews** | **Required checks** |
|----|:--:|:--:|----|
| release/demo/\* | ❌ Blocked | 2 approvals | Build, Sonar gate, Grype gate, smoke test |
| release/uat/\* | ❌ Blocked | 1 approval | Build, Sonar gate, Grype gate |
| release/dev/\* | ❌ Blocked | 1 approval | Build, Sonar gate |
| feature/\*, fix/\*, hotfix/\* | ✅ Allowed author-only | N/A | Unit test before PR, where configured |

## **7. Cleanup Policy**

- Delete feature/\*, fix/sonar-\*, fix/vapt-\* branches immediately after merge.

- hotfix/\* branches deleted after back-merge to uat and dev is confirmed.

- Old release/dev/x.y.z / release/uat/x.y.z branches archived (not deleted) once superseded, for audit traceability — retain last 3 versions minimum.

## **8. Quick Reference**

release/dev/1.0.0 → DEV pipeline  
release/uat/1.0.0 → UAT pipeline  
release/demo/1.0.0 → PROD (Demo) pipeline  

feature/\* → off release/dev, merges to release/dev  
fix/sonar-\* → off release/dev, merges to release/dev, delete after merge  
fix/vapt-\* → off release/dev (or active env if urgent), promotes forward  
hotfix/\* → off release/demo, back-merge to uat + dev
