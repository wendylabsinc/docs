# Cloud Functional Requirements

## 1. Organisations

1.1 Every resource in the cloud belongs to exactly one organisation.  
1.2 A user may be a member of multiple organisations.

---

## 2. Identity and Access

### 2.1 Roles

2.1.1 The cloud defines three membership roles: owner, admin, and member.  
2.1.2 An owner may perform any action within the organisation, including transferring ownership and removing members.  
2.1.3 An admin may manage devices, applications, deployments, and members.  
2.1.4 A member may view and interact with devices, applications, and deployments.  
2.1.5 An organisation has exactly one owner at all times. Ownership may be transferred to any existing member.

### 2.2 Personal Access Tokens

2.2.1 Users may create personal access tokens for programmatic access.  
2.2.2 A token carries the same permissions as the user who created it.  
2.2.3 Tokens may be revoked at any time.

### 2.3 Third-party Integrations

2.3.1 Third-party services may be granted delegated access on behalf of a user.  
2.3.2 The user approves the scope of access explicitly.  
2.3.3 Delegated access may be revoked at any time without affecting the user's own credentials.

---

## 3. Device Provisioning

### 3.1 Enrollment Tokens

3.1.1 Authorised users may generate enrollment tokens scoped to a specific device and organisation.  
3.1.2 Tokens are time-limited and single-use.

### 3.2 Certificate Issuance

3.2.1 A device presents an enrollment token to receive a digital identity certificate from the cloud.  
3.2.2 The certificate binds the device to its organisation and is used to authenticate all subsequent cloud communication.  
3.2.3 The cloud records the certificate and its association to the device.

### 3.3 Certificate Renewal

3.3.1 Devices may renew their certificate before expiry without human intervention.

### 3.4 Revocation

3.4.1 Authorised users may revoke a device certificate at any time.  
3.4.2 A revoked device is immediately denied access to the cloud.

---

## 4. Device Management

### 4.1 Fleet Overview

4.1.1 The cloud presents all provisioned devices in an organisation with their connection status and metadata.

### 4.2 Device Detail

4.2.1 Each device exposes its installed applications, their running state, the time it last reported, and any active deployments targeting it.

### 4.3 Device Metadata

4.3.1 Devices carry a name, location, tags, and hardware specifications.  
4.3.2 Authorised users may update device metadata.

### 4.4 Targeting

4.4.1 Devices may be targeted individually, by tag, by metadata field, or as an entire fleet.

---

## 5. Application Management

### 5.1 Applications

5.1.1 An application has a name and belongs to an organisation.  
5.1.2 An application is the parent record for all its releases and deployments.

### 5.2 Releases

5.2.1 A release is a named, immutable version of an application referencing a specific built artifact.

### 5.3 Deployments

5.3.1 A deployment targets a set of devices and specifies a release and desired running state: running, stopped, or absent.  
5.3.2 Devices receive and apply deployment instructions asynchronously.  
5.3.3 The cloud tracks the convergence state of each targeted device independently.

### 5.4 Deployment Status

5.4.1 For each targeted device, the cloud reports whether the instruction was delivered and whether the device has applied it.  
5.4.2 The cloud records the time the instruction was last delivered and the time the device last reported its state.

### 5.5 Ad-hoc Control

5.5.1 Authorised users may start, stop, or remove a specific application on a specific device outside of a versioned deployment.

---

## 6. Observability

### 6.1 Log Collection

6.1.1 The cloud receives and retains log streams from devices.  
6.1.2 Each log entry is attributed to its originating device and application.

### 6.2 Log Tailing

6.2.1 Authorised users may follow the live log output of an application on a specific device in real time.

### 6.3 Metrics

6.3.1 The cloud receives and retains metrics from device applications.  
6.3.2 Authorised users may query metrics over time.

### 6.4 Notifications

6.4.1 The cloud notifies users of significant events including application crashes, devices going offline, and deployments failing to converge.  
6.4.2 Notification events and delivery channels are defined in a separate document.

---

## 7. Audit

7.1 The cloud maintains an append-only audit log of all user actions.  
7.2 Each entry records the actor, the action, and the timestamp.  
7.3 No user may modify or delete audit log entries.  
7.4 Authorised users may query the audit log for their organisation.
