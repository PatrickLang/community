<!--

Reviewing this doc

- Questions & comments
Please feel free to put them inline as a quote attributed to yourself:

> [name=Patrick Lang] This sounds great!


- Edits & clarifications
If it makes the doc better, go for it!


- Change management
I have another copy of this in my own git repo, and will be manually merging changes back into the final. I'm planning to cover questions in the main text of the article, and remove comments before taking it to other SIGs.

-->
Windows Service Accounts in Kubernetes
======================================

**Authors**: Patrick Lang ([@patricklang](https://www.github.com/patricklang)), Ryan Puffer ([@rpsqrd](https://www.github.com/rpsqrd))


*What is Active Directory?*

Windows applications and services typically use Active Directory identities to facilitate authentication and authorization between resources. In a traditional virtual machine scenario, the computer is joined to an Active Directory domain which enables it to use Kerberos, NTLM, and LDAP to identify and query for information about other resources in the domain. 

*What is a service account?*

A Group Managed Service Account (gMSA) is a shared Active Directory identity that enables common scenarios such as authenticating and authorizing incoming requests and accessing downstream resources such as a database server, file share, or other workload.

*How is it applied to containers?*

To achieve the scale and speed required for containers, Windows uses a group managed service account in lieu of individual computer accounts to enable Windows containers to communicate with Active Directory.

Different containers on the same machine can use different gMSAs to represent the specific workload they are hosting, allowing operators to granularly control which resources a container has access to. However, to run a container with a gMSA identity, an additional parameter must be supplied to the Windows Host Compute Service to indicate the intended identity. This proposal seeks to add support in Kubernetes for this parameter to enable Windows containers to communicate with other enterprise resources.



## Parts of solution

### Provisioning Workflow

This is a workflow that's handled outside of Kubernetes until the last step. Most organizations have tight control over who can create Active Directory accounts, and the role is usually separated from that of an application owner. Performing these steps in Kubernetes is not a goal.

1. An Active Directory domain administrator:
    a. Joins all Windows nodes to the Active Directory domain.
    b. Provisions the service account account, and gives details to application admin. 
    c. Assigns access to a security group to control what machines can use this service account. This group should include all authorized Kubernetes nodes.
2. The application admin uses a [script](https://github.com/MicrosoftDocs/Virtualization-Documentation/tree/live/windows-server-container-tools/ServiceAccounts) to verify the account exists, and capture enough data to uniquely identify it into a JSON file. This doesn't contain any passwords or crypto secrets.
3. The application admin stores this JSON in the Kubernetes secret store with `k8s create secret generic WindowsServiceAccount1 --from-file=credspec.json`

### Deployment Workflow

Deployment should be as simple as using a Kubernetes secret, but a different property will be used that's specific to Windows. This is optional, and would only be used in machines already joined to an Active Directory domain.

> [name='patricklang']TODO: Need to complete this deployment manifest after taking a closer look at APIs

``` yaml
deployment:
    pod:
        security:
            WindowsCredentialSpec = 'WindowsServiceAccount1'
        
```


### Node workflow

No extra steps or configuration are needed in the kubelet config, and no additional daemon sets are required. The deployment should contain everything needed as its passed from the apiserver. Conceptually, this is similar to an imagePullSecret in that it's read from the host, not passed directly to a pod or container.

1. Deployment request includes a container with WindowsCredentialSpec set to secret name
2. The CRI implementation passes the JSON contents of the secret to  windows.CredentialSpec per the [OCI Runtime Specification](https://github.com/opencontainers/runtime-spec/blob/master/config-windows.md#credential-spec)
> With dockershim, this intermediate step may be needed before 2. The kubelet copies the `secret` (that's not actually a secret) to a local file with a unique name under the folder `c:\programfiles\docker\credentialspecs` such as `d4d521e1-2933-4d59-9b8e-ebe51524a76e.json`. When the pod is destroyed, the file should be deleted by the CRI implementation.


## Glossary


* **Active Directory (AD)** - is a directory service built into Windows Server that provides identity and directory services using LDAP and enables the use of security protocols including Kerberos and X.509 public key infrastructure.
* **CredentialSpec** . A JSON object that controls which Windows-specific accounts are available to the container. This could eventually be extended to include additional Windows-specific accounts or certificate stores. [Microsoft docs](https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/manage-serviceaccounts)
* **Group Managed Service Account** (gMSA) - are a type of Active Directory account that makes it easy to secure services without sharing a password. Multiple machines or containers share the same gMSA as needed to authenticate connections between services. When you create a gMSA, you can choose what other accounts (computers and/or users) are able to use it.
