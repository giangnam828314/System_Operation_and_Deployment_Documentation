# 07 — Final Acceptance Checklist

## A. Server

- [ ] OS/version verified
- [ ] CPU/RAM/Disk verified
- [ ] Admin account verified
- [ ] sudo verified
- [ ] SSH custom port verified
- [ ] `PermitRootLogin no` verified
- [ ] Firewall verified

## B. Application stack

- [ ] Web Server installed and active
- [ ] PHP target version verified
- [ ] Required PHP extensions verified
- [ ] PHP-FPM active
- [ ] Database active
- [ ] DB connectivity verified
- [ ] Bitrix Environment/application configuration verified

## C. Backup / Restore

- [ ] External HDD mounted successfully
- [ ] Backup copy completed
- [ ] File size verified
- [ ] SHA-256/checksum verified where applicable
- [ ] Source code restored
- [ ] Database restored
- [ ] DB tables verified
- [ ] CRM data spot check passed
- [ ] Users spot check passed
- [ ] Deals spot check passed
- [ ] Tasks spot check passed
- [ ] Attachments/files spot check passed

## D. Permission

- [ ] Owner/Group verified
- [ ] `/upload/` write test passed
- [ ] `/bitrix/cache/` write test passed
- [ ] `/bitrix/managed_cache/` write test passed
- [ ] Upload test passed
- [ ] CRM create/update test passed
- [ ] No critical Permission Denied errors
- [ ] SELinux reviewed

## E. Domain / SSL

- [ ] Domain resolves correctly
- [ ] DNS points to target
- [ ] SSL certificate valid
- [ ] HTTPS working
- [ ] Redirect behavior verified

## F. Functional acceptance

| Test | Result | Evidence |
|---|---|---|
| Login | ☐ Pass ☐ Fail | __________ |
| CRM | ☐ Pass ☐ Fail | __________ |
| Deals | ☐ Pass ☐ Fail | __________ |
| Users | ☐ Pass ☐ Fail | __________ |
| Tasks | ☐ Pass ☐ Fail | __________ |
| Workflows | ☐ Pass ☐ Fail | __________ |
| File Upload | ☐ Pass ☐ Fail | __________ |
| File Download | ☐ Pass ☐ Fail | __________ |
| Database Read/Write | ☐ Pass ☐ Fail | __________ |
| Background jobs | ☐ Pass ☐ Fail | __________ |
| Email/notification (if in scope) | ☐ Pass ☐ Fail | __________ |

## G. Logs / Stability

- [ ] `systemctl --failed` reviewed
- [ ] Web Server log reviewed
- [ ] PHP-FPM log reviewed
- [ ] Database log reviewed
- [ ] No critical application error
- [ ] No unexplained permission error

## H. Cut-over

- [ ] Maintenance window recorded
- [ ] Final backup recorded
- [ ] Final restore recorded
- [ ] DNS switch recorded
- [ ] SSL validation recorded
- [ ] Actual downtime recorded
- [ ] User communication sent

## I. Rollback readiness

- [ ] Old system still available per retention plan
- [ ] Rollback trigger defined
- [ ] DNS rollback path tested/verified
- [ ] Decision owner identified
- [ ] Data reconciliation plan understood

## J. Sign-off

| Role | Name | Date/Time | Status |
|---|---|---|---|
| IT Executor | __________ | __________ | PASS / FAIL |
| IT Reviewer | __________ | __________ | PASS / FAIL |
| Customer / Y Việt | __________ | __________ | ACCEPT / REJECT |
| Final Approver | __________ | __________ | APPROVED / REJECTED |

## Evidence naming standard

```text
01-installation/
02-restore/
03-security/
04-acceptance/
06-troubleshooting/
```

Evidence files should be named with step number + purpose, for example:

```text
05-php-version.png
08-db-import.png
10-functional-crm.png
```
