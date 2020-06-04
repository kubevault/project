# AppRole Support in KubeVault

https://learn.hashicorp.com/vault/identity-access-management/approle


- [ ] Step 1: Enable AppRole auth method (Persona: admin)

This steps should be done via spec.authMethods in [VaultServer CRD](
https://github.com/kubevault/operator/blob/3535586316cb243c4b4648fbacf06b148da73c0c/apis/kubevault/v1alpha1/vaultserver_types.go#L89-L91)

- [ ] Step 2: Create a role with policy attached (Persona: admin)

 - Here the policy can be created using Vault cli or `VaultPolicy` CRD . No additional change is required.

 - To assign policies to an AppRole, we should add `AppRoleSubjectRef` field in [SubjectRef](https://github.com/kubevault/operator/blob/3535586316cb243c4b4648fbacf06b148da73c0c/apis/policy/v1alpha1/vaultpolicybinding_types.go#L78). This `AppRoleSubjectRef` can be same as [VaultAppRoleSpec](https://github.com/kotyara85/operator/blob/1a9712a3dbb3e0517a6a00b6e9b4b629e9ce9b82/apis/approle/v1alpha1/vaultapprole_types.go#L47). I don't think we need a new CRD for AppRole.

  One question is should we delete the vault AppRole when this `VaultPolicyBinding` CRD is deleted? We have not done that for `KubernetesSubjectRef` today. But we could do this. This will require adding a finalizer for `VaultPolicyBinding` to delete the Vault role before removing `VaultPolicyBinding` object.

- [ ] Step 3: Get RoleID and SecretID

Vault docs are not clear who performs this step. My idea will be that `admin` persona issues the SecretID and shares the info with `app` persona. The key question is how do we avoid unauthorized persons from creating such secrets.

I can think of 2 ways accomplishing this. We could potentially support both mechanisms:

*Using Secret*: We could follow the pattern used to [create additional api tokens for Kubernetes s/a](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#to-create-additional-api-tokens). So, the process will be:

  - `admmin` creates a new k8s secret with annotations `kubevault.com/vault-ref: <vault_appbinding_name>` and `kubevault.com/approle: <approle_name>`. We could support additional parameters via annotaion shown here: https://www.vaultproject.io/api-docs/auth/approle#generate-new-secret-id

  - KubeVault operator will watch such secrets and update the secret data will `role_id`, `secret_id` and `secret_id_accessor`

My general idea is that this secret should be created in the same namespace where Vault AppBinding resides and only `admin` personas should have access to this namespace. To give `app` personas access, `admin` persona will create a `Role` & `RoleBinding` with a resoure name matching this secret. Creating the `Role` and `RoleBinding` is outside the scope of KubeVault operator (I think?).

*Using a new AppRoleSecretIDRequest CRD*: Here the user (`app` persona) initiates the request, `admin` personas `approve/deny` the request. If approved, KubeVault will create a secret with the `role_id`, `secret_id` and `secret_id_accessor`. It will be up to the user to ensure that no one else can read this secret.

The benefit with this approach is that the `AppRoleSecretIDRequest` object can be created in any namespace. The secret will be created in the same namespace as `AppRoleSecretIDRequest` object. So, user can run a pod by mounting this secret and can perform additional operations inside the pod using `AppRole`s.


- [ ] Step 4: Login with RoleID & SecretID (Persona: app)

The idea here is KubeVault operator connects to an external Vault server using AppRole in the AppBinding CRD. To support this we need to add
`AppRole *AppRoleConfig` in [VaultServerConfiguration](https://github.com/kubevault/operator/blob/3535586316cb243c4b4648fbacf06b148da73c0c/apis/config/v1alpha1/vault_server_config_types.go#L28)

The info from AppBinding should be used to implement `AuthInterface` interface in `approle` package here: https://github.com/kubevault/operator/tree/3535586316cb243c4b4648fbacf06b148da73c0c/pkg/vault/auth

- [ ] Step 5: Read secrets using the AppRole token
- [ ] Response Wrap the SecretID

No special support needed for this in KubeVault operator. One place this can be supported is in CSI driver. Currently CSI driver uses Kubernetes Auth method and pod's service account to read secrets from Vault and mount into the pod. If the application that is running inside the pod, instead connects to Vault directly to read secrets, we could extend the CSI driver to inject a wrapped SecretID into the pod. My question is now necessary is this?

- [ ] Limit the SecretID Usages

This should be supported by the PolicyBinding
