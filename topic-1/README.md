# Topic 1: Cisco ISE Guest Authentication - Wired Access Control with Sponsor-Based Account Creation

## Research Summary

This document provides a comprehensive analysis of Cisco ISE guest authentication for wired access, focusing on sponsor-based guest account creation workflows. The research draws from Cisco's official documentation, community deployment guides, and practitioner experience.

---

## Table of Contents

1. [Overview](#overview)
2. [How Cisco ISE Handles Guest Authentication for Wired Access](#how-cisco-ise-handles-guest-authentication-for-wired-access)
3. [Guest Sponsor Workflow](#guest-sponsor-workflow)
4. [Configuration Steps for Wired Guest Access](#configuration-steps-for-wired-guest-access)
5. [Authentication Methods for Wired Guests](#authentication-methods-for-wired-guests)
6. [Best Practices](#best-practices)
7. [Troubleshooting](#troubleshooting)
8. [References](#references)

---

## Overview

Cisco ISE (Identity Services Engine) provides guest access capabilities that allow visitors, contractors, consultants, and customers to securely access a controlled network segment without full corporate credentials. The framework combines a captive portal, authentication policy, and authorization profile to determine what a guest can access and for how long.

ISE ships with three default guest portal types:

- **Hotspot Portal** - No username/password required; guest accepts an Acceptable Use Policy (AUP) or enters an access code
- **Self-Registered Guest Portal** - Visitor creates their own account with name, email, and company
- **Sponsored Guest Portal** - An employee (sponsor) creates the guest account on the visitor's behalf

For wired guest access with sponsor-based account creation, the **Sponsored Guest Portal** is the primary model, though self-registration with sponsor approval is also supported.

---

## How Cisco ISE Handles Guest Authentication for Wired Access

### Central Web Authentication (CWA) - The Preferred Architecture

The recommended architecture for Cisco ISE guest access is **Central Web Authentication (CWA) with Guest Flow**. This approach centralizes the entire web-auth process on ISE rather than pushing portal configuration to individual switches.

#### The Two-Stage Guest Flow Process

Guest-access authorization with ISE happens in two distinct stages:

**Stage 1: MAC Authentication Bypass (MAB) Redirect**
1. The guest device connects to a wired port on the switch
2. The switch initiates MAB authentication with ISE
3. ISE returns a redirect Access-Control List (ACL) instead of full access
4. The NAD (Network Access Device) sends the guest's browser traffic to the ISE portal over HTTPS on port 8443
5. At this stage, the network device and ISE track the endpoint with a common session ID

**Stage 2: Authenticated Session Merge**
1. The guest submits credentials on the portal (provided by the sponsor)
2. ISE validates the credentials against the guest user database
3. ISE sends a Change of Authorization (CoA) to the switch
4. The NAD re-authorizes the session and applies the final access policy
5. The original MAB session and the newly authenticated session merge into one tracked session

This session merge depends on a session identifier that ties the initial MAB event to the later authenticated one. If the NAD does not support proper session accounting, the merge breaks, and guests may experience portal loops.

### Why CWA Beats Local Web Authentication (LWA)

- **Centralized management**: All portal configuration lives on ISE, not on every switch
- **Simplified maintenance**: Portal updates happen once on ISE, not on every NAD
- **Better certificate handling**: Single certificate management point
- **Consistent user experience**: Same portal behavior regardless of which switch the guest connects to

LWA (Local Web Authentication) still has a place where a NAD cannot perform CoA, for small isolated sites, or where legacy hardware does not support the redirect ACL model.

---

## Guest Sponsor Workflow

### Sponsor Account Types and Permissions

ISE supports three default sponsor groups that control who can create guest accounts:

| Sponsor Group | Permission Level |
|---|---|
| **ALL_ACCOUNTS** (default) | Sponsors can manage all guest user accounts |
| **GROUP_ACCOUNTS** (default) | Sponsors can manage only guest accounts created by sponsors from the same group |
| **OWN_ACCOUNTS** (default) | Sponsors can manage only the guest accounts they created |

Sponsors can be:
- **Internal ISE user accounts** defined under Administration > Identity Management > Identities > Users
- **Active Directory accounts** - ISE joins AD and maps AD groups to sponsor groups
- **LDAP accounts** - Similar to AD integration

### Sponsor Portal Access

Sponsors access the Guest Management portal via one of these methods:

1. **Portal Test URL** - Default option, can be sent to sponsors for bookmarking
2. **Sponsor Portal FQDN** - An easy-to-remember URL requiring DNS configuration (recommended)
3. **ISE Admin GUI** - Direct access through the ISE administration interface

The Sponsor Portal URL format is typically:
```
https://sponsorportal.yourcompany.com
```
or
```
https://ise-ip:8443/sponsorportal/PortalSetup.action?portal=portalID
```

### Guest Account Creation Workflow (Sponsored Flow)

**Step 1: Sponsor Logs In**
- Sponsor opens the Sponsor Portal URL
- Enters username and password (AD credentials or internal ISE account)
- Accepts AUP if required by policy

**Step 2: Create Guest Account**
- Sponsor navigates to the "Create Accounts" tab
- Selects the Guest Type from dropdown (e.g., "Contractor", "Daily", "Weekly", or custom types)
- Enters guest information:
  - First Name, Last Name (or system generates random username)
  - Email address (optional, for credential delivery)
  - Phone number (optional, for SMS delivery)
  - Company name (optional)
  - Duration of access (start/end times, respecting the guest type limits)
  - Number of simultaneous devices allowed
- The system generates a username and password based on configured policies

**Step 3: Credential Delivery**
Credentials can be delivered to the guest via:
- **Email** - Requires SMTP configuration in ISE
- **SMS/Text** - Requires SMS gateway configuration
- **Print** - Sponsor can print credentials for hand-delivery
- **Verbal** - Sponsor reads credentials to the guest

**Step 4: Guest Connects and Authenticates**
- Guest plugs into a wired port or connects to the guest network
- Device is redirected to the ISE Guest Portal
- Guest enters the credentials provided by the sponsor
- Guest accepts the AUP (if configured)
- ISE validates credentials and sends CoA to the switch
- Guest receives network access per the authorization policy

**Step 5: Sponsor Manages Account**
Sponsors can perform ongoing management via the "Manage Accounts" tab:
- Edit and delete guest accounts
- Extend guest account duration
- Suspend guest accounts
- Reinstate expired guest accounts
- Resend and reset passwords for guests
- Approve pending accounts (for self-registration with sponsor approval)

### Guest Account Lifecycle States

| State | Description |
|---|---|
| **Created** | Account created but guest has not yet logged in through the portal |
| **Active** | Guest has successfully signed in through the portal, or bypassed it (for 802.1x supplicants) |
| **Suspended** | Sponsor has temporarily disabled the account |
| **Expired** | Account duration has ended |
| **Pending** | Self-registered account awaiting sponsor approval |

### 802.1x Guest Accounts via Sponsor Portal

An advanced use case allows sponsors to create guest accounts that authenticate via 802.1x (wired or wireless) rather than through the captive portal. This requires:

1. Creating a custom Guest Type with "Allow guest to bypass the Guest portal" enabled
2. Creating a custom Endpoint Identity Group for the guest devices
3. Configuring an Identity Source Sequence that includes the Guest Users database
4. Building a Policy Set with authentication and authorization rules for 802.1x
5. Assigning a downloadable ACL (dACL) that permits internet access while denying internal network access

When "Allow guest to bypass the Guest portal" is enabled, the guest account goes directly to "Active" state, skipping the "Awaiting Initial Login" state and AUP page. The guest can authenticate using credentials in a wired 802.1x supplicant without being redirected to a captive portal.

**Note**: Windows systems do not have wired 802.1x (Wired Autoconfig service) enabled by default. This service must be enabled for Windows devices to use wired 802.1x authentication.

---

## Configuration Steps for Wired Guest Access

### Phase 1: Network Prerequisites (Switch Configuration)

Before configuring ISE, the network access device must be properly configured.

#### 1.1 Switch Capabilities Required

- **Layer 3 SVI** for the guest network (routable interface for browser redirection)
- **IP device tracking** enabled (usually default, critical for endpoint tracking)
- **Switch management IP** for RADIUS communication with ISE
- **Global RADIUS and AAA configurations** for ISE integration

#### 1.2 Required Ports

| Protocol | Port | Purpose |
|---|---|---|
| HTTPS | TCP 8443 | Portal redirect traffic |
| RADIUS Authentication | UDP 1812 (or 1645) | Authentication requests |
| RADIUS Accounting | UDP 1813 (or 1646) | Session accounting |
| **CoA** | **UDP 1700** | **Change of Authorization (critical!)** |

#### 1.3 Example Catalyst Switch Configuration

```ios
! Enable HTTP server for URL redirection
ip http server

! Define RADIUS server (ISE)
radius server ISE-PSN
 address ipv4 <ISE-IP> auth-port 1812 acct-port 1813
 key <shared-secret>
 automate-tester probe-name ISE-Probe

! AAA configuration
aaa new-model
aaa authentication dot1x default group ISE
aaa authorization network default group ISE
aaa accounting dot1x default start-stop group ISE

! Enable CoA (Critical for guest flow)
aaa server radius dynamic-author
 client <ISE-IP> server-key <shared-secret>

! Global dot1x controls
dot1x system-auth-control
radius-server vsa send authentication
radius-server vsa send accounting

! Redirect ACL (name must match ISE Authorization Profile exactly)
ip access-list extended REDIRECT_ACL_CWA
 deny ip any host <ISE-PSN-IP-1>
 deny ip any host <ISE-PSN-IP-2>
 permit tcp any any eq www
 permit tcp any any eq 443
 permit udp any any eq domain
 deny ip any any

! Pre-auth ACL (optional, for low-impact mode)
ip access-list extended PRE-AUTH-ACL
 permit udp any any eq 68
 permit udp any any eq 67
 permit tcp any host <ISE-PSN-IP> eq 8443
 deny ip any any

! Access port configuration
interface GigabitEthernet1/0/1
 description Guest-Wired-Port
 switchport access vlan <guest-vlan>
 switchport mode access
 authentication open
 authentication order mab dot1x
 authentication priority dot1x mab
 authentication port-control auto
 authentication periodic
 authentication timer reauthenticate server
 authentication timer inactivity server
 authentication violation restrict
 mab
 dot1x pae authenticator
 dot1x timeout tx-period 10
 spanning-tree portfast
 spanning-tree bpduguard enable
```

#### 1.4 Key Switch Configuration Points

- **`authentication open`** - Enables monitor mode; port is open before authentication
- **`authentication order mab dot1x`** - Tries 802.1x first, falls back to MAB
- **`mab`** - Enables MAC Authentication Bypass
- **`dot1x pae authenticator`** - Enables 802.1x on the port
- **Redirect ACL** - Must be defined on the switch with the exact same name as configured in ISE Authorization Profile
- **CoA** (`aaa server radius dynamic-author`) - Required for ISE to re-authorize the session after guest login

### Phase 2: ISE Configuration

#### 2.1 Add Network Access Device to ISE

Navigate to **Administration > Network Resources > Network Devices**:
1. Click **Add**
2. Enter device name and IP address
3. Configure RADIUS Authentication Settings with the shared secret
4. Click **Submit**

#### 2.2 Portal Certificate Configuration

For each ISE PSN hosting a Guest Portal:
- Use a **wildcard certificate** issued from a **Public Certificate Authority**
- Navigate to **Administration > System > Certificates > Certificate Signing Request**
- Complete CSR and submit to Public CA
- Import the signed certificate
- Bind the certificate to the Guest Portal role under **Certificates > System Certificates**

**Critical**: Use a public CA for guest portals. Guest devices are unmanaged and have no reason to trust an internal CA. A missing intermediate certificate will cause failures on iOS and Android devices.

#### 2.3 Configure Guest Portal

Navigate to **Work Centers > Guest Access > Portals & Components > Guest Portals**:

For a **Sponsored Guest Portal**:
1. Click **Create** and select **Sponsored-Guest Portal** as the portal type
2. Provide Portal Name and Description
3. Configure Portal Settings:
   - Ports, Ethernet interfaces, certificate group tags
   - Identity source sequences
   - Authentication method
4. Configure page-specific settings:
   - **Login Page Settings**: Guest credential and login guidelines
   - **AUP Page Settings**: Acceptable Use Policy configuration
   - **Guest Device Registration Settings**: Auto-register or manual registration
   - **Post-Login Banner Page Settings**: Additional information for guests
   - **Authentication Success Settings**: Post-authentication redirect
   - **Support Information Page Settings**: Help desk troubleshooting info
5. Customize the portal theme (branding, logo, colors)
6. Test the portal URL directly from the configuration page

#### 2.4 Configure Guest Types

Navigate to **Work Centers > Guest Access > Portals & Components > Guest Types**:

ISE ships with three built-in guest types:
- **Contractor** - Extended access, up to a year
- **Daily** - 1 to 5 days of access
- **Weekly** - A couple of weeks of access

For custom guest types, configure:
- **Endpoint identity group** for guest device registration
- **Allow guest to bypass the Guest portal** (for 802.1x supplicants)
- **Session duration** and **login time limits**
- **Account validity** (from first login or sponsor-specified date)
- **Endpoint purge schedule** (default: 30 days)

#### 2.5 Configure Sponsor Groups

Navigate to **Work Centers > Guest Access > Portals & Components > Sponsor Groups**:

1. Edit or create a sponsor group
2. Configure **Members**:
   - Add internal ISE user groups
   - Add Active Directory groups (requires AD join)
   - Add LDAP groups
3. Configure permissions:
   - Which guest types sponsors can assign
   - Account management capabilities
   - Access to ERS REST API (for automation)

#### 2.6 Configure Sponsor Portal

Navigate to **Work Centers > Guest Access > Portals & Components > Sponsor Portals**:

1. Select the default Sponsor Portal (or create a new one)
2. Configure **Portal Settings**:
   - Set the **Fully Qualified Domain Name (FQDN)** (e.g., `sponsor.yourcompany.com`)
   - This is the easy-to-remember URL for sponsors
3. Customize the portal theme
4. Update DNS to resolve the FQDN to the ISE IP address

#### 2.7 Configure SMTP and SMS (Optional but Recommended)

Navigate to **Settings > SMTP Server** and **SMS Gateway**:
- Configure SMTP server for email credential delivery
- Configure SMS gateway for text message credential delivery
- This eliminates manual handoff of credentials

#### 2.8 Configure Downloadable ACL (dACL)

Navigate to **Policy > Policy Elements > Authorization > Downloadable ACLs**:

**Pre-authentication dACL** (applied before guest login):
```
permit tcp any host <ISE-PSN-IP-1> eq 8443
permit tcp any host <ISE-PSN-IP-2> eq 8443
permit udp any any eq 68
permit udp any any eq 67
deny ip any 10.0.0.0 0.255.255.255
deny ip any 172.16.0.0 0.15.255.255
deny ip any 192.168.0.0 0.0.255.255
permit ip any any
```

**Post-authentication dACL** (applied after successful login):
```
permit udp any any eq 68
permit udp any any eq 67
permit udp any any eq 53
permit tcp any any eq 53
deny ip any 10.0.0.0 0.255.255.255
deny ip any 172.16.0.0 0.15.255.255
deny ip any 192.168.0.0 0.0.255.255
permit ip any any
```

#### 2.9 Configure Authorization Profiles

Navigate to **Policy > Policy Elements > Results > Authorization > Authorization Profiles**:

**Redirect Authorization Profile** (for initial MAB session):
1. Create a new Authorization Profile (e.g., `Guest-Portal-Redirect`)
2. Set the **Downloadable ACL** to the pre-auth dACL
3. Configure **Web Redirection**:
   - Select **Centralized Web Auth**
   - Enter the **ACL name** (must match the switch redirect ACL exactly)
   - Select the **Guest Portal** (Sponsored-Guest Portal)
   - Enter the **Static IP/Host name/FQDN** of the ISE PSN
4. Save the profile

**Permit Access Authorization Profile** (for post-auth session):
1. Create a new Authorization Profile (e.g., `Guest-Internet-Access`)
2. Set the **Downloadable ACL** to the post-auth dACL
3. Save the profile

**Important**: Create separate Authorization Profiles for each PSN hosting the portal, using the FQDN of each PSN. This ensures session affinity - the PSN that owns the RADIUS session is the same PSN serving the portal.

#### 2.10 Configure Policy Sets

Navigate to **Policy > Policy Sets**:

**Authentication Policy**:
1. Expand the Default Policy Set (or create a new one)
2. Add an authentication rule matching MAB traffic from guest VLAN/ports
3. Set the identity source to include Guest Users database

**Authorization Policy**:
Create two authorization rules in sequence:

**Rule 1: Redirect to Guest Portal** (matches initial MAB session)
- **Condition**: `Wired_MAB` OR `Wireless_MAB` (for wired-only, use `Wired_MAB`)
- **Result**: The Redirect Authorization Profile (e.g., `Guest-Portal-Redirect`)

**Rule 2: Guest Access** (matches post-authentication session)
- **Condition**: `Guest_Flow` (built-in condition matching re-authenticated guest sessions)
- **Result**: The Permit Access Authorization Profile (e.g., `Guest-Internet-Access`)

**For Wired Guest Access**:
The built-in `WiFi_Redirect_to_Guest_Login` policy can be modified to match `Wired_MAB` with an OR condition, or duplicated as a separate rule for wired traffic.

### Phase 3: End-to-End Configuration Summary

| Component | Configuration Location | Key Settings |
|---|---|---|
| Switch | IOS/IOS-XE CLI | AAA, MAB, CoA, Redirect ACL, Port config |
| ISE Network Device | Administration > Network Resources | Device IP, shared secret |
| Portal Certificate | Administration > System > Certificates | Public CA wildcard cert, bound to Guest Portal |
| Guest Portal | Work Centers > Guest Access > Portals & Components | Sponsored-Guest Portal, AUP, customization |
| Guest Types | Work Centers > Guest Access > Portals & Components | Duration, endpoint group, bypass settings |
| Sponsor Groups | Work Centers > Guest Access > Portal & Components | Members (AD groups), permissions |
| Sponsor Portal | Work Centers > Guest Access > Portals & Components | FQDN, theme, customizations |
| dACLs | Policy > Policy Elements > Results > Authorization | Pre-auth and post-auth ACLs |
| Authorization Profiles | Policy > Policy Elements > Results > Authorization | Redirect profile and permit profile |
| Policy Sets | Policy > Policy Sets | Auth rules for MAB and Guest Flow |

---

## Authentication Methods for Wired Guests

### 1. MAB + Central Web Authentication (CWA) - Primary Method

This is the standard and recommended approach for wired guest access:

1. **Initial authentication**: Device triggers MAB on the switch port
2. **ISE response**: Returns redirect ACL, forcing browser traffic to ISE portal
3. **Guest login**: Guest enters sponsor-provided credentials on the portal
4. **CoA**: ISE sends Change of Authorization to the switch
5. **Final authorization**: Switch applies the post-auth dACL, granting internet access

**Advantages**:
- No supplicant required on the guest device
- Works with any device that has a web browser
- Centralized management on ISE
- Full audit trail of guest sessions

**Considerations**:
- CoA (UDP 1700) must be reachable between ISE and the switch
- Session merge depends on proper RADIUS accounting
- Portal certificate must be trusted by guest devices

### 2. 802.1X with Guest Credentials (Bypass Portal)

An alternative approach where guest accounts authenticate directly via 802.1x without captive portal redirection:

1. **Sponsor creates guest account** with "Allow guest to bypass the Guest portal" enabled
2. **Guest plugs into port** and 802.1x authentication initiates
3. **Guest enters credentials** in the 802.1x supplicant
4. **ISE validates** against the Guest Users database
5. **Authorization profile** applies the appropriate dACL

**Advantages**:
- Faster authentication (no portal redirect)
- Better security posture (802.1x encryption)
- Can use dACLs and SGTs for fine-grained access control

**Considerations**:
- Requires 802.1x supplicant on the guest device
- Windows requires "Wired Autoconfig" service enabled
- AUP acceptance must be handled separately (or skipped)
- Less suitable for walk-in guests with personal devices

### 3. Hotspot Portal (No Credentials)

For environments where credential management is not desired:

1. Guest connects to the network
2. Redirected to the Hotspot Portal
3. Guest accepts the AUP (or enters an access code)
4. CoA grants network access

**Advantages**:
- Simplest user experience
- No account management required
- Suitable for public areas (lobbies, cafeterias)

**Considerations**:
- No individual accountability
- Limited access control granularity
- Access codes can be shared

### 4. Self-Registered Guest Portal

Guests create their own accounts, optionally with sponsor approval:

1. Guest connects and is redirected to the Self-Registration Portal
2. Guest fills in registration form (name, email, company, etc.)
3. ISE creates the account (immediately or after sponsor approval)
4. Guest receives credentials and logs in

**Advantages**:
- Self-service reduces sponsor workload
- Can require sponsor approval for additional security
- Full audit trail of registration and access

**Considerations**:
- Higher friction than hotspot
- Registration data must be validated
- Sponsor approval adds delay

---

## Best Practices

### Network Design

1. **Use a dedicated guest VLAN** - Separate guest traffic from corporate traffic at Layer 2/3
2. **Place ISE PSN in DMZ** (optional) - For enhanced security, place the PSN serving guest portals in a DMZ
3. **Ensure CoA reachability** - This is the #1 point of failure; verify UDP 1700 before deployment
4. **Enable RADIUS accounting** - Required for session merge in Guest Flow
5. **Use multiple PSNs** - Deploy at least two PSNs for redundancy; configure session affinity via FQDN-based Authorization Profiles

### Portal Configuration

6. **Use public CA certificates** - Guest devices are unmanaged and will not trust internal CAs
7. **Test on mobile devices** - Certificate chain issues that never appear on Windows/macOS show up immediately on iOS and Android
8. **Configure AUP appropriately** - Display on first login only (not every session) to reduce friction
9. **Enable SMTP/SMS** - Automate credential delivery to reduce sponsor burden

### Sponsor Management

10. **Limit sponsor group scope** - Use GROUP_ACCOUNTS or OWN_ACCOUNTS to limit sponsor permissions
11. **Use AD integration** - Map AD groups to sponsor groups for centralized identity management
12. **Set appropriate guest type durations** - Match session timeouts to actual visit patterns, not defaults
13. **Configure endpoint purge** - Set endpoint purge schedules independently of account expiry

### Policy Design

14. **Use role-based access** - Create distinct guest types for different visitor categories (contractor vs. walk-in)
15. **Apply bandwidth rate limiting** - Use QoS attributes in Authorization Profiles to prevent bandwidth abuse
16. **Segment guest traffic** - Use downloadable ACLs to restrict access to only necessary resources
17. **Consider SGTs** - For Software-Defined Segmentation, apply Scalable Group Tags to guest sessions

### Pre-Deployment Validation

18. **Run a Proof of Concept (PoC)** - Test CoA, certificate trust, and sponsor workflows before production
19. **Validate with multiple device types** - Test on Windows, macOS, iOS, and Android
20. **Test failover scenarios** - Verify PSN failover works correctly
21. **Check RADIUS Live Logs** - Use Operations > RADIUS > Live Logs as primary diagnostic tool

---

## Troubleshooting

### Common Issues and Solutions

| Symptom | Likely Cause | First Check |
|---|---|---|
| Guest never sees portal page | Redirect ACL missing or not mapped | NAD ACL config and FlexConnect ACL binding |
| Portal loads but login fails silently | CoA not reaching NAD | UDP 1700 reachability, shared secret match |
| Certificate warning on mobile only | Incomplete certificate chain | ISE certificate store, intermediate cert binding |
| Guest authenticated but no network access | Session merge failed | RADIUS accounting status on NAD |
| Guest disconnected shortly after login | Endpoint purge or accounting stop mismatch | Endpoint purge settings, accounting interim updates |
| Portal loops after valid credentials | Session merge broken | RADIUS accounting messages not reaching ISE |

### Diagnostic Commands

**On the Switch:**
```ios
! Check authentication session details
show authentication sessions interface <interface>
show authentication sessions

! Verify AAA server status
show aaa servers

! Check CoA client configuration
show running-config | include aaa server radius dynamic-author
```

**On ISE:**
- Navigate to **Operations > RADIUS > Live Logs**
- Look for the MAB entry, redirect ACL push, authentication event, and CoA entry
- Click the magnifying glass on any row for full attribute dump

### Key Diagnostic Signatures in Live Logs

1. **MAB authentication entry** - Confirms device associated and hit the authentication policy
2. **Redirect ACL push** - Confirms NAD received the correct authorization profile
3. **Guest authentication event** - Confirms ISE processed credentials correctly
4. **CoA dynamic authorization entry** - Confirms session re-authorized and merged

A missing or failed CoA entry is the most reliable indicator of session-merge problems.

---

## References

### Cisco Official Documentation
1. [Cisco ISE 3.4 Admin Guide - Guest and Secure WiFi](https://www.cisco.com/c/en/us/td/docs/security/ise/3-4/admin_guide/b_ise_admin_3_4/b_ISE_admin_guest.html)
2. [Cisco ISE Guest Access Prescriptive Deployment Guide](https://community.cisco.com/t5/security-knowledge-base/ise-guest-access-prescriptive-deployment-guide/ta-p/3640475)
3. [ISE Guest Account Management](https://www.cisco.com/c/en/us/support/docs/security/identity-services-engine/215931-ise-guest-account-management.html)
4. [Sponsor Portal User Guide for Cisco ISE](https://www.cisco.com/c/en/us/td/docs/security/ise/sponsor_portal/sponsor-portal-user-guide-for-cisco-identity-services-engine.pdf)
5. [ISE Secure Wired Access Prescriptive Deployment Guide](https://community.cisco.com/t5/security-knowledge-base/ise-secure-wired-access-prescriptive-deployment-guide/ta-p/3641515)

### Community and Practitioner Resources
6. [Cisco ISE Guest Access: A Practical Configuration Playbook (Re-solution)](https://re-solution.co.uk/cisco-ise-guest-access/)
7. [ISE Wired Guest - Integrating IT](https://integratingit.wordpress.com/2020/01/19/ise-guest-access/)
8. [802.1x Guest Users Created via Sponsor Portal (ISE Support)](https://www.ise-support.com/2020/02/19/802-1x-guest-users-created-via-sponsor-portal/)
9. [ISE 3.0 Guest Access with Sponsored Guest - LabMinutes](http://www.labminutes.com/sec0342_ise_30_guest_access_sponsored_guest_1)

### Related Topics
10. [ISE Secure Wired Access Prescriptive Deployment Guide](https://community.cisco.com/t5/security-knowledge-base/ise-secure-wired-access-prescriptive-deployment-guide/ta-p/3641515)
11. [Cisco ISE 802.1x Wired Configuration Guide (PingLabz)](https://www.pinglabz.com/cisco-ise-802-1x-wired-configuration-guide/)

---

*Research compiled on: September 16, 2026*
*Source articles: 3 primary articles + 8 supplementary sources*
