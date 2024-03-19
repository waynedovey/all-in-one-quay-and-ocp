#Get the correct clients

wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.13.29/oc-mirror.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.13.29/openshift-install-linux.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.13.29/openshift-client-linux.tar.gz

tar -xvf oc-mirror.tar.gz
tar -xvf openshift-install-linux.tar.gz
tar -xvf openshift-client-linux.tar.gz

mv oc-mirror openshift-install oc kubectl /usr/local/bin/
chmod -R a+x /usr/local/bin/ 

#SSL Cert Creation

openssl req -new -newkey rsa:2048 -nodes -keyout registry.digitaldovey.net.key -out registry.digitaldovey.net.csr

openssl s_client -connect registry.digitaldovey.net:8443 -servername registry.digitaldovey.net -showcerts

#sign and request cert registry.digitaldovey.net.pem

cp registry.digitaldovey.net.pem /etc/pki/ca-trust/source/anchors
sudo update-ca-trust extract

#Quay Mirror Guide

hostnamectl hostname registry.digitaldovey.net

Get a pull secret https://console.redhat.com/openshift/install/pull-secret

/root/pull_secret.txt

{
  "auths": {
    "registry.digitaldovey.net:8443": {
      "auth": "aW5pdDpXOWlLVUxWZDZHT2tKTzJ4M0IrajFGSjYzN21YQjA1VnFpeWg0Z0xURm5hSE0xVmIwajJ0d2xNbk95ZFlJNnp6",
      "email": ""
    },
    "cloud.openshift.com": {
      "auth": "",
      "email": "wdovey@redhat.com"
    },
    "quay.io": {
      "auth": "=",
      "email": "wdovey@redhat.com"
    },
    "registry.connect.redhat.com": {
      "auth": "",
      "email": "wdovey@redhat.com"
    },
    "registry.redhat.io": {
      "auth": "",
      "email": "wdovey@redhat.com"
    }
  }
}

wget https://developers.redhat.com/content-gateway/rest/mirror/pub/openshift-v4/clients/mirror-registry/latest/mirror-registry.tar.gz

wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/openshift-client-linux.tar.gz

./mirror-registry install --quayHostname registry.digitaldovey.net --targetHostname registry.digitaldovey.net  --quayRoot /var/mirror-registry --initPassword changeme --ssh-key /root/.ssh/id_rsa --sslCert /root/registry.digitaldovey.net.pem

export OCP_RELEASE="4.13.29"
export LOCAL_REGISTRY="registry.digitaldovey.net:8443"
export LOCAL_REPOSITORY="ocp4/openshift4"
export PRODUCT_REPO="openshift-release-dev"
export LOCAL_SECRET_JSON="/root/pull_secret.txt"
export RELEASE_NAME="ocp-release"
export ARCHITECTURE="x86_64"  

oc adm release mirror -a ${LOCAL_SECRET_JSON} --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE} --to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} --to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE}

--> install-config.yaml

imageContentSources:
- mirrors:
  - registry.digitaldovey.net:8443/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - registry.digitaldovey.net:8443/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev

### Operator Mirroring Guide ###

wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/latest/oc-mirror.tar.gz
gunzip oc-mirror.tar.gz
tar -xvf oc-mirror.tar
chmod +x oc-mirror
mv oc-mirror /usr/local/bin/

dnf install docker -y

mkdir .docker/
cat ./pull_secret.txt | jq . > pull_secret.json
cp pull_secret.json .docker/config.json
sudo docker login registry.redhat.io

sudo docker run -p50051:50051 -it registry.redhat.io/redhat/redhat-operator-index:v4.14

curl -sSL "https://github.com/fullstorydev/grpcurl/releases/download/v1.8.7/grpcurl_1.8.7_linux_x86_64.tar.gz" | sudo tar -xz -C /usr/local/bin
grpcurl -plaintext localhost:50051 api.Registry/ListPackages > redhat-operators-packages.out

oc-mirror list operators --catalog registry.redhat.io/redhat/redhat-operator-index:v4.13 --package compliance-operator

cat <<EOF > imageSetConfig.yaml
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v1alpha2
archiveSize: 4
storageConfig:
  local:
    path: /var/mirror-registry/operators/
mirror:
  operators:
    - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.13
      packages:
        - name: compliance-operator
          channels:
          - name: stable
        - name: file-integrity-operator
EOF


oc mirror --config=/root/imageSetConfig.yaml file:///var/mirror-registry/operators/

cd /var/mirror-registry/operators/

docker login -u init -p changeme registry.digitaldovey.net:8443 

oc-mirror --from=/var/mirror-registry/operators/ docker://registry.digitaldovey.net:8443/ocp/operators

cd /var/mirror-registry/operators/oc-mirror-workspace/results-1710538892

cat <<EOF > catalogSource-redhat-operator-index.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: redhat-operator-index-0
  namespace: openshift-marketplace
spec:
  image: registry.digitaldovey.net:8443/ocp/operators/redhat/redhat-operator-index:v4.13
  sourceType: grpc
EOF

cat <<EOF > imageContentSourcePolicy.yaml
---
apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  labels:
    operators.openshift.org/catalog: "true"
  name: redhat-operators-0
spec:
  repositoryDigestMirrors:
  - mirrors:
    - registry.digitaldovey.net:8443/ocp/operators/openshift4
    source: registry.redhat.io/openshift4
  - mirrors:
    - registry.digitaldovey.net:8443/ocp/operators/compliance
    source: registry.redhat.io/compliance
EOF

oc patch operatorhubs/cluster --type merge --patch \
 '{"spec":{"sources":[{"disabled": true,"name": "community-operators"},{"disabled": true,"name": "certified-operators"},{"disabled": true,"name": "redhat-marketplace"},{"disabled": true,"name": "redhat-operators"}]}}'

#oc delete catalogsources -n openshift-marketplace --all
#oc delete imagecontentsourcepolicy --all

oc apply -f /var/mirror-registry/operators/oc-mirror-workspace/results-1710538892
