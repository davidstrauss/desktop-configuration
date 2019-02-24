# Desktop Management Utilities

This document catalogs potential management utilities for a deployment of Fedora-based desktops.

## Evaluation Criteria

### Requirements
* Reporting on enrolled machine compliance
* Reporting on applied security updates

### Pros
* Operable against SaaS (e.g. hosted Chef, Azure AD) or lightweight cloud services (S3)
* Can remediate security configuration, not just audit
* Has provisioning-friendly design to accelerate new machine setup by IT
* Can configure setups like GPG smart card support and GPG-SSH agent integration
* Can support Windows or Mac desktops as well
* Used by other organizations to management developer desktops

### Cons
* Requires deployment of a complex, self-managed server (e.g. FreeIPA)

## Utilities

### Fleet Commander

[Homepage](https://fleet-commander.org/)

Fleet Commander provides desktop management in a form similar to Microsoft's Group Policy tools. However, because it stores the data differently than Group Policy, it cannot rely on a standard Windows Server (or Azure AD Domain Services) deployment; it must use FreeIPA. FreeIPA also supports Windows.

### Chef

[Homepage](https://www.chef.io/)

### OpenSCAP

[Guide for Fedora](https://static.open-scap.org/ssg-guides/ssg-fedora-guide-index.html)

### SSSD with Active Directory

[Documentation](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/windows_integration_guide/sssd-ad)
