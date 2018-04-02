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
> [name=Peter Hornyack]"in lieu of individual computer accounts"
Are individual accounts supported in addition to gMSAs if desired? Or will containers never be able to "run as" an individual AD user?
> [name=Patrick Lang]Not an individual user, at least at this point. Supporting additional account types would be a new feature in Windows and I can't commit to that in the near future.

Different containers on the same machine can use different gMSAs to represent the specific workload they are hosting, allowing operators to granularly control which resources a container has access to. However, to run a container with a gMSA identity, an additional parameter must be supplied to the Windows Host Compute Service to indicate the intended identity. This proposal seeks to add support in Kubernetes for this parameter to enable Windows containers to communicate with other enterprise resources.
> [name=Peter Hornyack]"an additional parameter must be supplied to the Windows Host Compute Service"
Does Docker on Windows already support the use of this parameter?
> [name=Patrick Lang]Yes, but it requires dropping a file on the node and referencing it by name: https://github.com/MicrosoftDocs/Virtualization-Documentation/blob/live/windows-server-container-tools/ServiceAccounts/README.md#running-containers . I clarified how dockershim would need to handle this under "Node Workflow" below.


## Parts of solution

### Provisioning Workflow

This is a workflow that's handled outside of Kubernetes until the last step. Most organizations have tight control over who can create Active Directory accounts, and the role is usually separated from that of an application owner. Performing these steps in Kubernetes is not a goal.

1. An Active Directory domain administrator:
    a. Joins all Windows nodes to the Active Directory domain.
    b. Provisions the service account and gives details to application admin. 
    c. Assigns access to a security group to control what machines can use this service account. This group should include all authorized Kubernetes nodes.
2. The application admin uses a [script](https://github.com/MicrosoftDocs/Virtualization-Documentation/tree/live/windows-server-container-tools/ServiceAccounts) to verify the account exists, and capture enough data to uniquely identify it into a JSON file. This doesn't contain any passwords or crypto secrets.
3. The application admin [stores](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-secret-generic-em-) this JSON in the Kubernetes secret store with `kubectl create secret generic WindowsServiceAccount1 --from-file=credspec.json`
> [name=Peter Hornyack]It sounds like there would be value in integrating these provisioning steps into kubeadm or whatever other tools will be used to manage clusters / add new nodes to them. Maybe not at first, but do you see this making sense at some point?
> [name=Patrick Lang]Yeah, I think this could fit into the realm of things kubeadm could do. It would be useful to have a cluster admin-level tool:
>  - Be able to append a new node to the security group as it's joined to the cluster
>  - Compare the list of nodes to the security group membership to check for omissions or old nodes no longer in use
> 
> Would there be any comparable tasks for Linux? I'm not too familiar with how LDAP is used in practice on Linux.

Example credentialspec.json

```json
{
  "CmsPlugins": [
    "ActiveDirectory"
  ],
  "DomainJoinConfig": {
    "DnsName": "contoso.com",
    "Guid": "244818ae-87ca-4fcd-92ec-e79e5252348a",
    "DnsTreeName": "contoso.com",
    "NetBiosName": "DEMO",
    "Sid": "S-1-5-21-2126729477-2524075714-3094792973",
    "MachineAccountName": "WebApplication1"
  },
  "ActiveDirectoryConfig": {
    "GroupManagedServiceAccounts": [
      {
        "Name": "WebApplication1",
        "Scope": "DEMO"
      },
      {
        "Name": "WebApplication1",
        "Scope": "contoso.com"
      }
    ]
  }
}
```


### Deployment Workflow

Deployment should be as simple as using a Kubernetes secret, but a different property will be used that's specific to Windows. This is optional, and would only be used in machines already joined to an Active Directory domain.


``` yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: iis-deployment
spec:
  replicas: 3
  template:
    metadata:
      labels:
        name: iis
    spec:
      containers:
      - name: iis
        image: microsoft/iis:windowsservercore-1709
        ports:
        - containerPort: 80
        securityContext:
          WindowsCredentialSpec: WindowsServiceAccount1
      nodeSelector:
        beta.kubernetes.io/os: windows        
```


> [name=Peter Hornyack]It looks like K8s already has a "service accounts" concept that uses secrets: [Service Accounts Automatically Create and Attach Secrets with API Credentials](https://kubernetes.io/docs/concepts/configuration/secret/#service-accounts-automatically-create-and-attach-secrets-with-api-credentials), [Configure Service Accounts for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/). Is there a way that we can/should use this existing channel to pass the Windows gMSA information through?
> [name=Patrick Lang] Will take a closer look at those. I do think there's a scenario where both would be used at the same time. Here's a potential log gather/audit scenario
>  - Use a K8s service account to auth to the apiserver and gather container stats or logs
>  - Use the gMSA to authenticate to an existing on-premises database and ship the metrics and logs there

### Node workflow

No extra steps or configuration are needed in the kubelet config, and no additional daemon sets are required. The deployment should contain everything needed (except the contents of the secret) as its passed from the apiserver. The kubelet will automatically pull the contents of the credentialspec from the secret, which is the same flow used for in-tree volume plugins ([cephfs](https://github.com/kubernetes/kubernetes/blob/a1437feb18c1b1cf319607cd36cfd2865fba3a5a/pkg/volume/cephfs/cephfs.go#L106
), iscsi, rbd, ...) today.

1. Deployment request includes a container with WindowsCredentialSpec set to secret name
> [name=Peter Hornyack] I'm confused about why this proposal goes through the secrets store when the JSON itself "doesn't contain any passwords or crypto secrets" (above). Why not just pass WindowsCredentialSpec = <JSON value> directly in the deployment spec and skip the secrets part?
> [name=Patrick Lang] Fair point, we should consider both alternatives. Some of the customers using this today through Docker swarm don't ask developers to supply these gMSA details. My assumption was that storing it in the cluster as a secret would make it easier to read and share deployment specs between dev and production environments. 
> 
> I'm expecting these to be in the 400 - 2000 character range today and that could increase if more features are added later. I just added a sample above.

2. The kubelet will retrieve the credentialspec JSON from the secret and set `windows.CredentialSpec` per the [OCI Runtime Specification](https://github.com/opencontainers/runtime-spec/blob/master/config-windows.md#credential-spec)

3. The CRI implementation will pass this to Windows as appropriate.

   a. Dockershim will need to implement an intermediate step because Docker EE-basic 17.x builds available on Windows read credentialspecs from files. The dockershim will need to copy the contents of the credentialspec to a local file with a unique name under the folder `c:\programfiles\docker\credentialspecs` such as `d4d521e1-2933-4d59-9b8e-ebe51524a76e.json`. When the pod is destroyed, the file should be deleted by dockershim. 

   b. CRI-containerd should be able to pass the JSON as-is to Windows


## Glossary


* **Active Directory (AD)** - is a directory service built into Windows Server that provides identity and directory services using LDAP and enables the use of security protocols including Kerberos and X.509 public key infrastructure.
* **CredentialSpec** . A JSON object that controls which Windows-specific accounts are available to the container. This could eventually be extended to include additional Windows-specific accounts or certificate stores. [Microsoft docs](https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/manage-serviceaccounts)
* **Group Managed Service Account** (gMSA) - are a type of Active Directory account that makes it easy to secure services without sharing a password. Multiple machines or containers share the same gMSA as needed to authenticate connections between services. When you create a gMSA, you can choose what other accounts (computers and/or users) are able to use it.
