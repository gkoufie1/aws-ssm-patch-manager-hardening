# Lab Notes - Patch Manager (account kwesi, us-east-1, 2026-10-12)

## Fleet
| Name | Instance ID | OS |
|---|---|---|
| lab-linux-al2023 | i-00f7bd1ca228005cc | Amazon Linux 2023 (latest AMI) |
| lab-linux-ubuntu | i-05d4e2ec6da19e296 | Ubuntu 24.04 (2026-04-09 AMI) |
| lab-win-1 | i-089bb3c4542d1e2eb | Windows Server 2022 (2026-06-10 AMI) |
| lab-win-2 | i-02744862b1bf386ba | Windows Server 2022 (2026-06-10 AMI) |

## BEFORE - initial Scan (all AWS default baselines)
| Node | Baseline | Installed | Missing | Failed | Security non-compliant |
|---|---|---|---|---|---|
| lab-linux-al2023 | pb-05c9c9bf778d4c4d0 | 65 | 0 | 0 | 0 |
| lab-linux-ubuntu | pb-0c7e89f711c3095f4 | 58 | 160 | 0 | 160 |
| lab-win-1 | pb-09ca3fb51f0412ec3 | 3 | 2 | 0 | 3 |
| lab-win-2 | pb-09ca3fb51f0412ec3 | 3 | 2 | 0 | 3 |

**Total missing: 164 patches across 3 of 4 nodes.**

## Linux install (Phase 4)
- Ubuntu: 160 missing -> 0 missing (installed count 58 -> 227). AL2023: still 0 missing.
- Troubleshooting: Patch now was submitted twice (19:28 and 19:36 EDT). The second run failed on Ubuntu with
  "Cannot acquire lock on /var/log/amazon/ssm/patch-baseline-concurrent.lock. Another patching operation is in progress."
  Root cause: concurrent execution while the first install was still running. The first run completed Success at ~19:40.
  No action needed beyond letting the first run finish; the lock prevented conflicting apt operations.

## Custom baseline (Phase 5)
- WindowsServerSecurityUpdates = pb-04fd4fb9d22180ff8
- WindowsServer2022 | SecurityUpdates + CriticalUpdates | Critical + Important | 0-day auto-approve | compliance level CRITICAL
- Unapproved available security updates reported as Noncompliant

## Windows install (Phase 7)
- Both Windows nodes installed KB5122882 (2026-09 Cumulative Update, Critical) and KB5126149 (2026-09 .NET CU, Important). Missing 2 -> 0, 0 failed.
- lab-win-1 used custom baseline pb-04fd4fb9d22180ff8 (via Patch Group=WindowsProd); lab-win-2 used AWS default pb-09ca3fb51f0412ec3. Routing confirmed.
- Finding: both still NON_COMPLIANT. Cause: KB5127069 (2026-09 Windows Recovery Environment Update) is classified
  SecurityUpdates but has severity "None", so the Critical/Important rule doesn't approve it. Because the baseline reports
  unapproved available security updates as Noncompliant, it surfaced instead of being silently ignored.

- Attempt 1: added KB5127069 to the baseline's Approved patches (compliance level Critical) and reran Install on lab-win-1
  (command 2bcf09d1). Result: still "available security updates not approved by the baseline: KB5127069".
- Root cause: KB5127069 is NOT in Patch Manager's Windows patch catalog (describe-available-patches returns nothing for it),
  so baseline approval rules/exceptions can't match it. Windows Update reports it, Patch Manager can't approve it.
- Attempt 2: out-of-band install via Run Command (AWS-RunPowerShellScript + Windows Update Agent API), then rescan.

- Attempt 2 result: Run Command installed KB5127069 on lab-win-1 (ResultCode 2 = Succeeded, no reboot). Rescan -> COMPLIANT.

## Automation (Phase 9)
- Maintenance window mw-08e56746b49343a12 "patch-windowsprod-weekly": cron(0 3 ? * SUN *) America/New_York, 2h duration, 1h cutoff.
- Target: tag:Patch Group = WindowsProd. Task: AWS-RunPatchBaseline, Operation=Install, RebootOption=RebootIfNeeded, concurrency 1, error threshold 1.
- Caught during CLI verification: task was first registered with the wrong document (AWS-ApplyAnsiblePlaybooks, sample
  hello-world playbook) and "Allow unregistered targets" was enabled. Both fixed before the first scheduled run.

## AFTER
| Node | Baseline | Missing | Failed | Status |
|---|---|---|---|---|
| lab-linux-al2023 | pb-05c9c9bf778d4c4d0 (AWS default) | 0 | 0 | COMPLIANT |
| lab-linux-ubuntu | pb-0c7e89f711c3095f4 (AWS default) | 0 (was 160) | 0 | COMPLIANT |
| lab-win-1 | pb-04fd4fb9d22180ff8 (custom, WindowsProd) | 0 (was 2) | 0 | COMPLIANT |
| lab-win-2 | pb-09ca3fb51f0412ec3 (AWS default, control) | 0 (was 2) | 0 | NON_COMPLIANT (KB5127069 left on purpose) |

**Before: 1/4 compliant, 164 missing patches. After: 0 missing, 0 failed; 3/4 compliant, with lab-win-2 kept as an unremediated control.**

## Cleanup (Phase 10)
- Maintenance window, custom baseline + patch group, and CloudFormation stack deleted. Verified via CLI: stack gone, 4 instances terminated,
  0 EBS volumes, 0 EIPs, SG and IAM role removed, 0 managed nodes. Run time ~6.5h, estimated cost ~$1.40.
