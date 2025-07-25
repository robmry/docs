---
description: Learn how to manage Single Sign-On for your organization or company.
keywords: manage, single sign-on, SSO, sign-on, docker hub, admin console, admin, security
title: Manage single sign-on
linkTitle: Manage
aliases:
- /admin/company/settings/sso-management/
- /single-sign-on/manage/
- /security/for-admins/single-sign-on/manage/
---

{{< summary-bar feature_name="SSO" >}}

## Manage organizations

> [!NOTE]
>
> You must have a [company](/admin/company/) to manage more than one organization.

{{% admin-sso-management-orgs product="admin" %}}

## Manage domains

{{< tabs >}}
{{< tab name="Admin Console" >}}

{{% admin-sso-management product="admin" %}}

{{< /tab >}}
{{< tab name="Docker Hub" >}}

{{% include "hub-org-management.md" %}}

{{% admin-sso-management product="hub" %}}

{{< /tab >}}
{{< /tabs >}}

## Manage SSO connections

{{< tabs >}}
{{< tab name="Admin Console" >}}

{{% admin-sso-management-connections product="admin" %}}

{{< /tab >}}
{{< tab name="Docker Hub" >}}

{{% include "hub-org-management.md" %}}

{{% admin-sso-management-connections product="hub" %}}

{{< /tab >}}
{{< /tabs >}}

## Manage users

> [!IMPORTANT]
>
> SSO has Just-In-Time (JIT) Provisioning enabled by default unless you have [disabled it](../provisioning/just-in-time/#sso-authentication-with-jit-provisioning-disabled). This means your users are auto-provisioned to your organization.
>
> You can change this on a per-app basis. To prevent auto-provisioning users, you can create a security group in your IdP and configure the SSO app to authenticate and authorize only those users that are in the security group. Follow the instructions provided by your IdP:
>
> - [Okta](https://help.okta.com/en-us/Content/Topics/Security/policies/configure-app-signon-policies.htm)
> - [Entra ID (formerly Azure AD)](https://learn.microsoft.com/en-us/azure/active-directory/develop/howto-restrict-your-app-to-a-set-of-users)
>
> Alternatively, see the [Provisioning overview](../provisioning/_index.md) guide.


### Add guest users when SSO is enabled

To add a guest that isn't verified through your IdP:

1. Sign in to [Docker Home](https://app.docker.com/) and select
your organization.
1. Select **Members**.
1. Select **Invite**.
1. Follow the on-screen instructions to invite the user.

### Remove users from the SSO company

To remove a user:

1. Sign in to [Docker Home](https://app.docker.com/) and select
your organization.
1. Select **Members**.
1. Select the action icon next to a user’s name, and then select **Remove member**, if you're an organization, or **Remove user**, if you're a company.
1. Follow the on-screen instructions to remove the user.

## Manage provisioning

Users are provisioned with Just-in-Time (JIT) provisioning by default. If you enable SCIM, you can disable JIT. For more information, see the [Provisioning overview](../provisioning/_index.md) guide.

## What's next?

- [Set up SCIM](../provisioning/scim.md)
- [Enable Group mapping](../provisioning/group-mapping.md)

