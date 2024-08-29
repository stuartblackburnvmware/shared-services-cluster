# Guide to Install Shared Services Cluster
A k8s cluster will be installed to run shared services such as Hashicorp Vault, Harbor, etc.

## Prereqs
* TKGS is deployed with AVI
* TKGS namespace created to be used for this new cluster
* bitnamicharts project created on Harbor instance

## Tools
* Tanzu CLI
    * TMC plugin
    * Apps Plugin
* yq
* ytt

## Prep values file
Copy the values template file and prep modify the values as needed
```
cp tanzu-cli/values/values-template.yml tanzu-cli/values/values.yml
```

## Create cluster group in TMC
```
ytt --data-values-file tanzu-cli/values -f tanzu-cli/cluster-group/cg-template.yml | tanzu tmc clustergroup create -f-
```

## Create cluster in TMC
Replace PROFILE with the name of the shared-services cluster as needed
```
export PROFILE=site1-ss
ytt --data-values-file tanzu-cli/values --data-value profile=$PROFILE -f tanzu-cli/clusters/cluster-template.yml > generated/$PROFILE-cluster.yml
tanzu tmc cluster create -f generated/$PROFILE-cluster.yml
```

## Enable helm and flux on Cluster Group
Replace CLUSTERGROUP with the name of the cluster group as needed
```
export CLUSTERGROUP=shared-services
tanzu tmc continuousdelivery enable -s clustergroup -g $CLUSTERGROUP
tanzu tmc helm enable -g $CLUSTERGROUP -s clustergroup
```

## Bootstrap cluster with gitops
This step configures the cluster group to use this git repo as the source for flux, specifically the flux folder. The gitops setup is done at the cluster group level so we don't need to individually bootstrap every cluster.

Before creating the TMC objects, you will need to rename the folders in flux/clusters to match your cluster names. Also if your cluster group name is different than shared-services you will need to rename the folder in flux/clustergroups along with the paths in the flux/clustergroups/<group-name>/base.yml.

### Create the gitrepo in TMC

This step is done manually for now since the current version of TMC-SM doesn't support everything we need. TODO: Automate this

Under your cluster group->Add-ons->Repository Credentials, create an SSH key repository credential

Note: Known hosts can be retrieved by running a `ssh-keyscan <git-server>` against your git server

Under your cluster group->Add-ons->Git repositories, add a Git repository pointing to your repo. Select the repository credential created in the previous step.

Once this is automated, it could be added with a command similar to the following:
```
ytt --data-values-file tanzu-cli/values -f tanzu-cli/cd/git-repo-template.yml > generated/gitrepo.yml
tanzu tmc continuousdelivery gitrepository create -f generated/gitrepo.yml -s clustergroup
```

### Create kustomizations
Create the base kustomization that will bootstrap the clusters and setup any initial infra. Kustomizations are split into pre and post in order to support deploying items which are dependent on other items.
```
ytt --data-values-file tanzu-cli/values -f tanzu-cli/cd/kust-template.yml > generated/kust.yml
tanzu tmc continuousdelivery kustomization create -f generated/kust.yml -s clustergroup
```

At this point clusters should start syncing in multiple kustomizations. You can check their status using the below command. there will be some in a failed state until the TAP install is done.

```
kubectl get kustomizations -A
```

## Vault Installation
### Login to Newly Created Cluster
```
export CLUSTER=site1-ss
export WCP=cluster01-wcp.stuart-lab.xyz
export CLUSTER_NAMESPACE=shared-services
kubectl vsphere login --tanzu-kubernetes-cluster-name $CLUSTER --server $WCP --tanzu-kubernetes-cluster-namespace $CLUSTER_NAMESPACE --insecure-skip-tls-verify
```

### Install vault air-gapped via helm
On machine with Internet access. Replace VAULTVERSION with the desired version. 1.2.1 used in this example.
```
export VAULTVERSION=1.2.1
helm dt wrap oci://docker.io/bitnamicharts/vault --version $VAULTVERSION
```

Copy vault-1.2.1.wrap.tgz to machine in air-gapped environment and run the following
NOTE: Ensure the helm dt plugin version you're unwrapping with matches the version it was wrapped with. Otherwise you may get errors.
NOTE: If `helm dt` is difficult to install on the air-gapped server, you can also use the standalone `dt`

```
export HARBOR=<harbor-fqdn>
export VAULTVERSION=1.2.1
helm dt unwrap vault-$VAULTVERSION.wrap.tgz oci://$HARBOR/bitnamicharts --insecure --yes
ytt --data-values-file tanzu-cli/values -f tanzu-cli/vault/override-values.yml > generated/override-values.yml
helm install vault oci://$HARBOR/bitnamicharts/vault -n vault --insecure-skip-tls-verify -f generated/override-values.yml
```

### Initialize vault
Run the following command to initialize vault. Make note of the unseal keys and initial root token.
```
kubectl exec --stdin=true --tty=true --namespace=vault vault-server-0 -- vault operator init
```
Unseal vault server. Repeat this command 3 times and using 3 different unseal keys
```
kubectl exec --stdin=true --tty=true --namespace=vault vault-server-0 -- vault operator unseal
```
### Install vault cli
Download the desired version of the vault CLI from https://developer.hashicorp.com/vault/downloads on a machine with Internet access

Copy the download zip file to the air-gapped jump server and extract it. Copy the extracted `vault` file to `/usr/local/bin`

### Login to vault server
```
export VAULT_ADDR="http://<vault-fqdn>" #TO DO - CHANGE TO HTTPS
export VAULT_TOKEN="<from previous step>"
vault login -tls-skip-verify
```

### Enable TLS
The following block of code in the helm data values is referencing a TLS secret by default:
```
    extraTls:
    - hosts:
        - #@ data.values.vault_fqdn
      secretName: #@ data.values.vault_fqdn + "-tls"%   
```

The TLS secret is managed manually. It can be created with the following command:
```
kubectl create secret tls <vault-fqdn>-tls \
  --cert=tls.crt \
  --key=tls.key \
  --namespace=vault
```

Once that secret is created, vault should be accessible over https by default

Note: Any time this certificate needs to be rotated, this secret will need to be updated.

### Create secrets engine
```
vault secrets enable -path=secret -tls-skip-verify -version=2 kv
```

### Add/read test secret to newly created secrets engine
```
vault kv put -tls-skip-verify secret/test-secret testkeyname=testkeyvalue
vault kv list -tls-skip-verify secret
vault kv get -tls-skip-verify secret/test-secret
```

### Configure vault AD integration
Run the following command to enable ldap auth:
```
vault auth enable ldap
```

Run the following command to configure ldap auth. Replace parameters as needed.
```
vault write auth/ldap/config \
    url="ldaps://dc01.stuart-lab.com" \
    userdn="ou=Users,ou=tmcsm,dc=stuart-lab,dc=com" \
    groupdn="ou=Groups,ou=tmcsm,dc=stuart-lab,dc=com" \
    binddn="CN=sa_pinniped,OU=tmcsm,DC=stuart-lab,DC=com" \
    bindpass='<replaceme>' \
    userattr="sAMAccountName" \
    groupfilter="(&(objectClass=group)(cn=tmc-admin)(member={{.UserDN}}))" \
    groupattr="cn" \
    insecure_tls=false \
    certificate=@dc-ca.pem
```

Configure an admin level access policy which will be assigned to the group defined in the step above:
```
vault policy write admin-policy - <<EOF
path "sys/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
path "auth/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
path "secret/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
EOF
```

Assign this policy to the desired AD group:
```
vault write auth/ldap/groups/tmc-admin policies=admin-policy
```