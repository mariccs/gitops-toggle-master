# toggle-master-gitops

Repositório só com manifestos Kubernetes — nenhum código de aplicação, nenhum Dockerfile. O ArgoCD, rodando no cluster EKS, sincroniza o que está aqui automaticamente. O pipeline de CI do repositório principal (`toggle-master`) atualiza a tag da imagem em `apps/<serviço>/deployment.yaml` a cada build, e o ArgoCD detecta essa mudança e faz o rollout sozinho.

Secrets não ficam aqui — eles continuam sendo aplicados fora do Git, via `k8s/generate-secrets.sh` no repositório principal.

## Estrutura

```
apps/
  auth-service/        deployment.yaml, service.yaml
  flag-service/
  targeting-service/
  evaluation-service/    + hpa.yaml
  analytics-service/     + hpa.yaml, keda-scaledobject.yaml (alternativa opcional)
shared/
  namespace.yaml
  configmap.yaml
  serviceaccounts.yaml
  ingress.yaml
argocd/
  applications/           uma Application por serviço + uma para os recursos compartilhados
```

## Configuração inicial (uma vez só)

1. Crie este repositório no GitHub (vazio) e faça o push deste conteúdo.

2. Substitua `<seu-usuario>` pela sua conta em todos os arquivos de `argocd/applications/*.yaml` (campo `repoURL`).

3. Substitua `<AWS_ACCOUNT_ID>` e `<AWS_REGION>` em cada `apps/*/deployment.yaml` pelos valores reais (saem do `terraform output` do repositório principal):
```bash
export AWS_ACCOUNT_ID=123456789012
export AWS_REGION=us-east-1
find apps -name deployment.yaml -exec sed -i.bak \
  "s#<AWS_ACCOUNT_ID>#$AWS_ACCOUNT_ID#g; s#<AWS_REGION>#$AWS_REGION#g" {} \;
find apps -name "*.bak" -delete
git add -A && git commit -m "chore: preenche registry do ECR" && git push
```

4. Substitua os ARNs de IRSA em `shared/serviceaccounts.yaml` (saem de `terraform output irsa_evaluation_service_role_arn` e `irsa_analytics_service_role_arn` no repositório principal).

5. No cluster, aponte o ArgoCD pra esse repositório aplicando as Applications:
```bash
kubectl apply -f argocd/applications/shared.yaml
kubectl apply -f argocd/applications/auth-service.yaml
kubectl apply -f argocd/applications/flag-service.yaml
kubectl apply -f argocd/applications/targeting-service.yaml
kubectl apply -f argocd/applications/evaluation-service.yaml
kubectl apply -f argocd/applications/analytics-service.yaml
```

6. Gere um Personal Access Token no GitHub com permissão de escrita neste repositório, e cadastre no repositório principal como secret `GITOPS_REPO_TOKEN` (e `GITOPS_REPO` com o valor `<seu-usuario>/toggle-master-gitops`) — é o que o pipeline de CI usa pra commitar a tag nova aqui.

Depois disso, o fluxo é: push no `toggle-master` → pipeline builda, escaneia, publica no ECR e atualiza a tag aqui → ArgoCD sincroniza sozinho no cluster.
