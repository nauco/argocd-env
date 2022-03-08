# argocd

* <https://argo-cd.readthedocs.io/en/stable/getting_started/>
* <https://argocd-applicationset.readthedocs.io/en/stable/Getting-Started/>

## create eks cluster

* <https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/create-cluster.html>

## Install addons 

* alb-ingress-controller 
    * <https://kubernetes-sigs.github.io/aws-load-balancer-controller/v1.1/guide/controller/setup/>
* external-dns
    * <https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/guide/integrations/external_dns/>  

## generate values.yaml & ingress.yaml

1. argocd admin password 를 잊어버리지 않기 위해, aws ssm 에 저장합니다.
2. github 계정으로 인증하기 위해 client id 와 client secret 을 저장 합니다.
3. github org (jenana-devops) 에 team (psa) 을 만들고 권한을 부여 합니다.
4. alb 생성을 위해 ingress 생성합니다. 

```bash
#################
# for htpasswd
#################
sudo apt-get install apache2-utils -y
sleep 5

#####################
# variables
#####################
export ARGOCD_HOSTNAME="REPLACE_ME" # argocd.example.com

export GITHUB_ORG="jenana-devops"
export GITHUB_TEAM="psa"

export ADMIN_PASSWORD="REPLACE_ME"
export ARGOCD_PASSWORD="$(htpasswd -nbBC 10 "" ${ADMIN_PASSWORD} | tr -d ':\n' | sed 's/$2y/$2a/')" 

#export ARGOCD_NOTI_TOKEN="REPLACE_ME" # xoxp-xxxx <https://api.slack.com/apps>

export ARGOCD_GITHUB_ID="REPLACE_ME" # github OAuth Apps <https://github.com/organizations/opspresso/settings/applications>
export ARGOCD_GITHUB_SECRET="REPLACE_ME" # github OAuth Apps

export DOMAIN_NAME="REPLACE_ME"
export AWS_ACM_CERT="$(aws acm list-certificates --query "CertificateSummaryList[].{CertificateArn:CertificateArn,DomainName:DomainName}[?contains(DomainName,'${ARGOCD_HOSTNAME}')] | [0].CertificateArn" | jq . -r)"

########################################
# put aws ssm parameter store
########################################
aws ssm put-parameter --name /k8s/common/admin-password --value "${ADMIN_PASSWORD}" --type SecureString --overwrite | jq .
aws ssm put-parameter --name /k8s/common/argocd-password --value "${ARGOCD_PASSWORD}" --type SecureString --overwrite | jq .

#aws ssm put-parameter --name /k8s/common/argocd-noti-token --value "${ARGOCD_NOTI_TOKEN}" --type SecureString --overwrite | jq .

aws ssm put-parameter --name /k8s/${GITHUB_ORG}/argocd-github-id --value "${ARGOCD_GITHUB_ID}" --type SecureString --overwrite | jq .
aws ssm put-parameter --name /k8s/${GITHUB_ORG}/argocd-github-secret --value "${ARGOCD_GITHUB_SECRET}" --type SecureString --overwrite | jq .


#########################################
# get aws ssm parameter store
#########################################
export ADMIN_PASSWORD=$(aws ssm get-parameter --name /k8s/common/admin-password --with-decryption | jq .Parameter.Value -r)
export ARGOCD_PASSWORD=$(aws ssm get-parameter --name /k8s/common/argocd-password --with-decryption | jq .Parameter.Value -r)
export ARGOCD_MTIME="$(date -u +"%Y-%m-%dT%H:%M:%SZ")"
export ARGOCD_GITHUB_ID=$(aws ssm get-parameter --name /k8s/${GITHUB_ORG}/argocd-github-id --with-decryption | jq .Parameter.Value -r)
export ARGOCD_GITHUB_SECRET=$(aws ssm get-parameter --name /k8s/${GITHUB_ORG}/argocd-github-secret --with-decryption | jq .Parameter.Value -r)

#export ARGOCD_NOTI_TOKEN=$(aws ssm get-parameter --name /k8s/common/argocd-noti-token --with-decryption | jq .Parameter.Value -r)

#######################################
# replace values.yaml & ingress.yaml
#######################################
cp values.yaml values.output.yaml
cp ingress.yaml argocd-ingress.yaml
find . -name values.output.yaml -exec sed -i "" -e "s/ARGOCD_HOSTNAME/${ARGOCD_HOSTNAME}/g" {} \;
find . -name values.output.yaml -exec sed -i "" -e "s@ARGOCD_PASSWORD@${ARGOCD_PASSWORD}@g" {} \;
find . -name values.output.yaml -exec sed -i "" -e "s/ARGOCD_MTIME/${ARGOCD_MTIME}/g" {} \;
find . -name values.output.yaml -exec sed -i "" -e "s/ARGOCD_GITHUB_ID/${ARGOCD_GITHUB_ID}/g" {} \;
find . -name values.output.yaml -exec sed -i "" -e "s/ARGOCD_GITHUB_SECRET/${ARGOCD_GITHUB_SECRET}/g" {} \;
find . -name values.output.yaml -exec sed -i "" -e "s/GITHUB_ORG/${GITHUB_ORG}/g" {} \;
find . -name values.output.yaml -exec sed -i "" -e "s/GITHUB_TEAM/${GITHUB_TEAM}/g" {} \;
find . -name argocd-ingress.yaml -exec sed -i "" -e "s/ARGOCD_HOSTNAME/${ARGOCD_HOSTNAME}/g" {} \;
find . -name argocd-ingress.yaml -exec sed -i "" -e "s@AWS_ACM_CERT@${AWS_ACM_CERT}@g" {} \;
```


## Install argoCD

> argoCD 를 설치 합니다.
* <https://artifacthub.io/packages/helm/argo/argo-cd>

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm search repo argo-cd

helm upgrade --install argocd argo/argo-cd -n argocd --create-namespace -f values.output.yaml
```

## Deploy ingress

> ingress 를 배포합니다.

```bash
kubectl apply -f argocd-ingress
```

## Check the result

> aws 에 alb 가 생성 되었습니다. route53 에서 host 와 연결되었는지 확인합니다. 

## argocd login

> argocd 에 로그인 합니다.
> cluster 를 add 합니다.

```bash
export ADMIN_PASSWORD=$(aws ssm get-parameter --name /k8s/common/admin-password --with-decryption | jq .Parameter.Value -r)

argocd login <ARGOCD_HOSTNAME> --grpc-web --username admin --password $ADMIN_PASSWORD --insecure

CONTEXT_NAME=`kubectl config view -o jsonpath='{.current-context}'`
argocd cluster add $CONTEXT_NAME --name eks-demo
argocd cluster list
```

## Deploy Application

> application 을 배포합니다.

```bash
kubectl apply -f https://raw.githubusercontent.com/jenana-devops/argocd-env/master/application.yaml
```

## Delete

> ingress 및 application 을 삭제합니다.

```bash
kubectl delete -f https://raw.githubusercontent.com/jenana-devops/argocd-env/master/application.yaml
kubectl delete -f argocd-ingress.yaml
```

> addons 와 eks cluster 삭제합니다. 

