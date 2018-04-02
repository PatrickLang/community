Windows Service Accounts in Kubernetes
======================================

**Authors**: Patrick Lang ([@patricklang](https://www.github.com/patricklang)), Ryan Puffer ([@rpsqrd](https://www.github.com/rpsqrd))
**Reviewers**: Peter Hornyack, more welcome!
*What is Active Directory?*

Windows applications and services typically use Active Directory identities to facilitate authentication and authorization between resources. In a traditional virtual machine scenario, the computer is joined to an Active Directory domain which enables it to use Kerberos, NTLM, and LDAP to identify and query for information about other resources in the domain. 

*What is a service account?*

A Group Managed Service Account (gMSA) is a shared Active Directory identity that enables common scenarios such as authenticating and authorizing incoming requests and accessing downstream resources such as a database server, file share, or other workload.

*How is it applied to containers?*

To achieve the scale and speed required for containers, Windows uses a group managed service account in lieu of individual computer accounts to enable Windows containers to communicate with Active Directory. As of right now, Windows cannot use individual Active Directory computer & user accounts.

Different containers on the same machine can use different gMSAs to represent the specific workload they are hosting, allowing operators to granularly control which resources a container has access to. However, to run a container with a gMSA identity, an additional parameter must be supplied to the Windows Host Compute Service to indicate the intended identity. This proposal seeks to add support in Kubernetes for this parameter to enable Windows containers to communicate with other enterprise resources.

It's also worth noting that Docker implements this in a different way that's not managed centrally. It requires dropping a file on the node and referencing it by name, eg: `docker run --credential-spec='file://foo.json'` . For more details see the Microsoft [doc](https://github.com/MicrosoftDocs/Virtualization-Documentation/blob/live/windows-server-container-tools/ServiceAccounts/README.md#running-containers).


## Parts of solution

### Provisioning Workflow

This is a workflow that's handled outside of Kubernetes until the last step. Most organizations have tight control over who can create Active Directory accounts, and the role is usually separated from that of an application owner. Performing these steps in Kubernetes is not a goal.

1. An Active Directory domain administrator:
    a. Joins all Windows nodes to the Active Directory domain.
    b. Provisions the service account and gives details to application admin. 
    c. Assigns access to a security group to control what machines can use this service account. This group should include all authorized Kubernetes nodes.
2. The application admin uses a [script](https://github.com/MicrosoftDocs/Virtualization-Documentation/tree/live/windows-server-container-tools/ServiceAccounts) to verify the account exists, and capture enough data to uniquely identify it into a JSON file. This doesn't contain any passwords or crypto secrets.
3. The application admin [stores](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-secret-generic-em-) this JSON in the Kubernetes secret store with `kubectl create secret generic WindowsServiceAccount1 --from-file=credspec.json`

Steps 2 and 3 could potentially be built into cluster management tools such as `kubeadm` at a later time for an easier cluster admin experience.

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

Today this would typically be around 400-2000 characters of JSON, but could grow in the future if additional account types or configurations are added.

### Deployment Workflow

Deployment should be as simple as using a Kubernetes secret, but a different property will be used that's specific to Windows. This is optional, and would only be used in machines already joined to an Active Directory domain. It has no relation or dependency on  existing K8s service accounts because it cannot be used inside of containers/pods to access any resources. It's used on the node side by the Windows runtime only.

There are two options proposed on how this could be done.

**Option 1** - Make the new field `WindowsCredentialSpec` a reference to a k8s secret. This keeps the verbose credential spec out of the deployment which may be easier to manage and read. If multiple deployments need the same credential spec and it needs to be changed, it can be done in a single place by updating the k8s secret. _The rest of this document will assume the workflow for Option 1._


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

**Option 2** - Make the new field `WindowsCredentialSpec` a literal. This will be JSON passed as-is in the OCI spec. If multiple deployments are using the same GMSA, each one will need to be updated individually. 

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
          WindowsCredentialSpec: >
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
      nodeSelector:
        beta.kubernetes.io/os: windows  
```




### Node workflow

No extra steps or configuration are needed in the kubelet config, and no additional daemon sets are required. The deployment should contain everything needed (except the contents of the secret) as its passed from the apiserver. The kubelet will automatically pull the contents of the credentialspec from the secret, which is the same flow used for in-tree volume plugins ([cephfs](https://github.com/kubernetes/kubernetes/blob/a1437feb18c1b1cf319607cd36cfd2865fba3a5a/pkg/volume/cephfs/cephfs.go#L106
), iscsi, rbd, ...) today.

1. Deployment request includes a container with WindowsCredentialSpec set to secret name

2. The kubelet will retrieve the credentialspec JSON from the secret and set `windows.CredentialSpec` per the [OCI Runtime Specification](https://github.com/opencontainers/runtime-spec/blob/master/config-windows.md#credential-spec)

3. The CRI implementation will pass this to Windows as appropriate.

   a. Dockershim will need to implement an intermediate step because Docker EE-basic 17.x builds available on Windows read credentialspecs from files. The dockershim will need to copy the contents of the credentialspec to a local file with a unique name under the folder `c:\programfiles\docker\credentialspecs` such as `d4d521e1-2933-4d59-9b8e-ebe51524a76e.json`. When the pod is destroyed, the file should be deleted by dockershim. 

   b. CRI-containerd should be able to pass the JSON as-is to Windows


## Glossary


* **Active Directory (AD)** - is a directory service built into Windows Server that provides identity and directory services using LDAP and enables the use of security protocols including Kerberos and X.509 public key infrastructure.
* **CredentialSpec** . A JSON object that controls which Windows-specific accounts are available to the container. This could eventually be extended to include additional Windows-specific accounts or certificate stores. [Microsoft docs](https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/manage-serviceaccounts)
* **Group Managed Service Account** (gMSA) - are a type of Active Directory account that makes it easy to secure services without sharing a password. Multiple machines or containers share the same gMSA as needed to authenticate connections between services. When you create a gMSA, you can choose what other accounts (computers and/or users) are able to use it.
