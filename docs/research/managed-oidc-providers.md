# Managed identity providers for the first private release

Research date: 2026-08-29

## Question and assumptions

Which managed identity provider best fits a small private application deployed to Cloud Run, while keeping setup and cost low and leaving room for user invitations, workspaces and future collaboration?

This comparison assumes:

- Next.js signs users in and NestJS/Fastify protects a GraphQL API;
- the first release has a small number of users and a monthly infrastructure target of roughly EUR 0–30;
- the product, not the identity provider, owns `Workspace`, `Membership`, invitation state and authorization rules;
- GitHub Actions deploys to GCP; authentication of that CI workload is separate from end-user authentication.

That third assumption is the most important portability boundary. OpenID Connect only guarantees that the pair `iss` + `sub` is a stable user identifier; email is explicitly not guaranteed to be stable or unique. The application should therefore keep its own `User` ID and an external-identity mapping `(issuer, subject)`, rather than use an IdP user ID or email as its domain ID ([OpenID Connect Core, claim stability](https://openid.net/specs/openid-connect-core-1_0.html#ClaimStability)).

## Short answer

For this release, **Google Identity Platform and Auth0 are the two strongest candidates, for different priorities**:

- Prefer **Google Identity Platform** when GCP cohesion, the local authentication emulator and the lowest likely cost matter most, and the team accepts Firebase client/Admin SDK integration plus application-owned invitations.
- Prefer **Auth0** when a conventional, discoverable OIDC integration and ready-made organization invitation flows matter more. Keep product workspaces in the application unless the five-organization free-plan ceiling is an intentional product limit.

**Okta is useful as a deliberate workforce-IAM learning choice, but is weaker on the stated low-cost criterion:** its Integrator Free Plan is limited to 10 active users and is positioned for building and testing integrations. **ZITADEL Cloud is the strongest standards-first alternative:** it has generous identity features and a self-hosting exit path, but its organization/project/grant model is more IAM than the product needs at the start.

This is a criteria-based shortlist, not the final provider decision. A small integration spike should make that decision after the initial authentication/session boundary is designed.

## Comparison

| Criterion | Google Identity Platform | Auth0 | Okta | ZITADEL Cloud |
| --- | --- | --- | --- | --- |
| Protocol surface | Issues OIDC-conformant ID tokens, but Google documents Firebase client and Admin SDKs as the normal integration surface | Full OAuth 2.0/OIDC flows and standard discovery endpoint | Full OAuth 2.0/OIDC app integration and discovery | Full OAuth 2.0/OIDC, standard discovery and PKCE |
| Local development | Best: official local Authentication Emulator, including test users and emulated third-party sign-in | `localhost` callbacks against a hosted tenant; no local IdP in the reviewed setup | `localhost` callbacks against a hosted Integrator org | Cloud instance supports HTTP localhost in development mode; an optional local runtime exists but is currently alpha |
| Invitations and membership | User administration and tenant silos exist; product-level invitations/workspace membership remain application work | Native Organizations, multi-organization membership, email invitations and organization roles | User activation email, groups and app assignment; these are workforce-directory concepts, not product workspaces | Native organizations and invitation email, but a user normally has one home organization and cross-org access uses project role assignments/grants |
| Small-release price | Social/email sign-in: first 50,000 MAU free; federating an external OIDC/SAML provider: only first 50 MAU free | Free: up to 25,000 MAU and 5 Organizations | Integrator Free Plan: 10 active users; paid Workforce Starter starts at USD 6/user/month billed annually | Free: 100 daily active users, unlimited stored users and organizations; Pro starts at USD 100/month |
| GCP/Cloud Run fit | Directly documented Cloud Run pattern; same project IAM, billing and Terraform provider | Cloud-neutral hosted service; application code on Cloud Run verifies standard tokens and stores separate vendor config | Cloud-neutral hosted service; separate tenant/config and potentially separate commercial contract | Cloud-neutral hosted service; separate instance/config |
| Infrastructure as code | Google Terraform provider covers Identity Platform configuration | Officially supported Auth0 Terraform provider | Official Okta Terraform provider | Official ZITADEL Terraform provider |
| Operational burden | Lowest vendor sprawl; emulator and GCP-native administration, but Firebase-specific SDK/config knowledge | Low; polished hosted login and Next.js SDK, but the free plan has one tenant and only one day of log retention | Medium; broad workforce controls and directory administration exceed the first release's needs | Medium; managed runtime is simple, but its IAM model and API surface are the richest of the shortlist |
| Switching cost if product authorization stays in-app | Medium: exported users are available, but application code uses Firebase token/SDK conventions | Low-to-medium for protocol code; medium for hosted credentials; high if Organizations, Actions or roles become product state | Low-to-medium for protocol code; high if groups, lifecycle and policies become product state | Low-to-medium for protocol code; lower infrastructure exit risk because self-hosting exists, but org/project grants are vendor-specific |

Pricing and plan limits above were checked on **2026-08-29** and are volatile; re-check the linked first-party pricing pages immediately before adoption or upgrade. Prices exclude taxes, support, SMS/phone authentication and possible minimum contract terms.

## Provider notes

### Google Identity Platform

Identity Platform is the most natural GCP choice. Google's Cloud Run guidance explicitly recommends it for end-user authentication and documents sending its ID token to a Cloud Run backend for verification ([Cloud Run end-user authentication](https://cloud.google.com/run/docs/authenticating/end-users), [end-to-end Cloud Run tutorial](https://cloud.google.com/run/docs/tutorials/identity-platform)). The Firebase Admin SDK verifies the tokens; Google also documents manual JWT verification and says the tokens conform to OIDC ([Admin Auth overview](https://firebase.google.com/docs/auth/admin), [ID-token verification](https://firebase.google.com/docs/auth/admin/verify-id-tokens)).

The trade-off is integration portability. Identity Platform can *consume* external OIDC providers through their discovery documents, while the documented application path uses Firebase SDKs and a project-specific token issuer/key-verification scheme ([OIDC federation](https://cloud.google.com/identity-platform/docs/web/oidc), [ID-token verification](https://firebase.google.com/docs/auth/admin/verify-id-tokens)). Replacing it is therefore more than changing an issuer URL, even though the resulting token is OIDC-shaped.

Local development is its clearest advantage. The official Authentication Emulator can create and edit test users, simulate email links and third-party sign-in, and make Admin SDKs accept emulator-issued unsigned tokens when explicitly configured ([Authentication Emulator](https://firebase.google.com/docs/emulator-suite/connect_auth)).

Identity Platform supports tenant-specific user silos and identity-provider configuration, aimed at B2B separation ([multi-tenancy](https://cloud.google.com/identity-platform/docs/multi-tenancy)). It also exposes privileged user creation and blocking `beforeCreate`/`beforeSignIn` functions, so an allowlist can prevent uninvited registration ([Admin user management](https://firebase.google.com/docs/auth/admin/manage-users), [blocking functions](https://cloud.google.com/identity-platform/docs/blocking-functions)). The reviewed first-party docs do not define an application-workspace invitation primitive; invitation records, acceptance and membership roles should remain in the application.

Current pricing makes direct email/social authentication free through 50,000 MAU. The much smaller free allowance of 50 MAU applies when Identity Platform federates an external OIDC or SAML provider ([pricing](https://cloud.google.com/identity-platform/pricing)). Configuration is available in the Google Terraform provider ([`google_identity_platform_config`](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/identity_platform_config)). User accounts can be exported, including Firebase SCRYPT password hashes subject to documented restrictions, which provides a migration path even though a target provider may not accept the same hash format ([Firebase auth export](https://firebase.google.com/docs/cli/auth)).

### Auth0

Auth0 exposes standard OAuth 2.0/OIDC flows and a conventional `/.well-known/openid-configuration` document, making it the cleanest match for a provider-neutral OIDC adapter ([flows](https://auth0.com/docs/get-started/authentication-and-authorization-flow), [discovery](https://auth0.com/docs/get-started/applications/configure-applications-with-oidc-discovery)). Its official Next.js quickstart supports the App Router and configures a hosted login flow for `localhost` ([Next.js quickstart](https://auth0.com/docs/quickstart/webapp/nextjs)); official local-testing guidance still uses the hosted Auth0 tenant rather than a local emulator ([local testing](https://auth0.com/docs/get-started/applications/work-with-auth0-locally)).

Auth0 Organizations are a close functional match for collaborative workspaces: a user may belong to multiple organizations, organization-scoped roles are supported, and the provider can send an invitation that creates or signs in the user and returns them to the application ([Organizations overview](https://auth0.com/docs/manage-users/organizations/organizations-overview), [organization invitations](https://auth0.com/docs/manage-users/organizations/configure-organizations/invite-members)). This convenience is also the largest lock-in risk. The free plan currently includes only five Organizations, while a product can naturally grow beyond five workspaces long before it reaches a meaningful MAU count ([pricing](https://auth0.com/pricing)). Consequently, use Auth0 Organizations only if provider-managed organization login is a chosen product feature; otherwise keep workspace invitations in the application.

The free plan currently lists up to 25,000 MAU, five Organizations, one custom domain and community support. It has one tenant, one day of log retention, and no separate production/development environments; the latter appears on paid Essentials ([pricing](https://auth0.com/pricing)). Auth0 has an officially supported Terraform provider ([Terraform provider](https://auth0.com/docs/deploy-monitor/auth0-terraform-provider)). User profiles have bulk export/import APIs, while password-hash and MFA-secret export requires a support case, eligibility review and organizational approval, so credential exit is possible but not frictionless ([user migration](https://auth0.com/docs/manage-users/user-migration), [credential export](https://auth0.com/docs/manage-users/user-migration/export-password-hashes-and-mfa-secrets)).

### Okta

Okta is a mature standards-based choice: it supports OIDC application integrations, Authorization Code with PKCE, standard token claims and groups, and exact localhost redirect URIs for development ([OIDC app integration](https://developer.okta.com/docs/guides/create-an-app-integration/openidconnect/main/), [SPA redirect setup](https://developer.okta.com/docs/guides/sign-into-spa-redirect/main/)). Local development still depends on a hosted Okta org.

Its onboarding model is workforce-oriented. Administrators create or stage users, activation can send a welcome link, groups organize users, and only assigned users/groups can authenticate to an application ([org setup](https://developer.okta.com/docs/guides/set-up-org/main/), [user activation API](https://developer.okta.com/docs/api/openapi/okta-management/management/tags/userlifecycle), [API Access Management](https://developer.okta.com/docs/concepts/api-access-management/)). This solves access to the application, not membership in arbitrary user-created workspaces; mapping Okta groups to product workspaces would couple the domain to the directory.

The Integrator Free Plan does not expire while active, but it is limited to 10 active users, deactivates after 180 days without a user sign-in, lacks paid support and is explicitly described as a developer/integration environment ([Integrator Free Plan limits](https://developer.okta.com/docs/reference/org-defaults/)). Paid Workforce Starter currently begins at USD 6 per user/month, billed annually; API Access Management is listed as an add-on ([Okta pricing](https://www.okta.com/pricing/)). This can fit a tiny private release, but the commercial boundary arrives much earlier and is less predictable than the other candidates.

Okta has a first-party Terraform provider and documentation for apps, groups and policies ([Terraform overview](https://developer.okta.com/docs/guides/terraform-overview/-/main/)). Protocol-level replacement is manageable if the application consumes only normalized OIDC claims. Using Okta groups, activation policies, hooks and directory lifecycle as domain state makes replacement substantially more involved; Okta's own migration guidance recommends keeping non-IAM data outside the identity platform ([migration planning](https://developer.okta.com/docs/guides/migrate-to-okta-plan/main/)).

### ZITADEL Cloud

ZITADEL is the standards-first alternative. It publishes normal OIDC discovery, recommends Authorization Code with PKCE and has a maintained Next.js example with localhost development mode ([OIDC endpoints](https://zitadel.com/docs/apis/openidoauth/endpoints), [Next.js example](https://zitadel.com/docs/sdk-examples/nextjs)). A new local-first runtime also exists, but the vendor labels it alpha, so it should not be a production dependency yet ([local preview](https://zitadel.com/next)).

Its free cloud plan currently includes 100 daily active users, unlimited stored users and organizations, and three identity providers; Pro starts at USD 100/month ([pricing](https://zitadel.com/pricing)). It also has an official Terraform provider and a documented path between cloud and self-hosted deployments ([Terraform provider](https://zitadel.com/docs/guides/manage/terraform-provider), [migration](https://zitadel.com/docs/guides/migrate/sources/zitadel)).

The main concern is domain fit, not capability. A ZITADEL organization is an IAM isolation boundary. A user normally belongs to one home organization; access across organizations is expressed through project role assignments and grants ([organizations](https://zitadel.com/docs/guides/manage/console/organizations-overview), [B2B model](https://zitadel.com/docs/guides/solution-scenarios/b2b)). ZITADEL can email a user invitation and let the user set up password, passkey or external SSO ([user onboarding](https://zitadel.com/docs/guides/manage/console/users-overview)), but using this model for lightweight, user-created product workspaces would bring more IAM semantics than the first release requires.

## GCP and GitHub Actions fit

The end-user IdP should not determine how GitHub Actions reaches GCP. GitHub's OIDC token can be exchanged through GCP Workload Identity Federation, avoiding a long-lived service-account key ([GitHub's GCP OIDC guide](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-google-cloud-platform), [Google authentication action](https://github.com/google-github-actions/auth)). The Cloud Run deployment action supports that flow directly ([Cloud Run deployment action](https://github.com/google-github-actions/deploy-cloudrun)). This works regardless of which of the four end-user providers is selected.

Google also recommends Identity-Aware Proxy for **internal** users of Cloud Run ([Cloud Run user authentication](https://cloud.google.com/run/docs/authenticating/end-users)). IAP is worth keeping as a possible outer gate for an internal-only preview, but it does not replace product-level users, workspaces, invitations or authorization and would not model future collaborators well. It should therefore not drive the application identity design.

## Switching-cost guardrails

The protocol is the cheap part of a provider switch. The expensive parts are credentials, sessions and vendor-specific workflow state. Apply these guardrails regardless of provider:

1. Give `User` an application-generated ID and store external identities as `(issuer, subject)`. Never identify a user by email.
2. Keep `Workspace`, `Membership`, invitation status and authorization policies in the application database.
3. Put OIDC/Firebase behavior behind one authentication boundary that emits a small internal principal such as `{ userId, sessionId }`; do not scatter raw vendor claims through GraphQL resolvers.
4. Treat provider roles/groups as optional login context, not the source of product authorization.
5. Store provider configuration declaratively where supported, and keep a tested user export procedure.
6. Expect any provider change to invalidate sessions and require at least some users to re-authenticate; password, MFA and passkey portability varies by provider and plan.

## Recommendation framed by criteria

Before making the final choice, implement the same thin spike with **Google Identity Platform and Auth0**:

`hosted/local sign-in -> Next.js session -> normalized internal principal -> authenticated GraphQL query -> sign-out`

Score it on setup time, local automated testing, amount of provider-specific code and the clarity of the Next.js-to-NestJS trust boundary.

- If the spike confirms that the Firebase emulator materially improves the feedback loop and application-owned invitations are acceptable, choose Google Identity Platform.
- If replacing issuer/client configuration in a generic OIDC library is materially cleaner and hosted-only local auth is acceptable, choose Auth0.
- Choose Okta only if learning or matching the anticipated company platform is promoted above the low-cost requirement.
- Choose ZITADEL only if standards, open-source/self-hosting optionality or its IAM organization model becomes an explicit goal.

Whichever candidate wins, **do not make provider-managed organizations the canonical workspace model in the first release**. That decision preserves both the product model and the realistic option to change providers later.
