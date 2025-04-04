## vault / consul backup

- create snapshot 

```bash
kubectl exec -n vault consul-consul-server-0 -- consul snapshot save /tmp/backup.snap
```

- copy snapshot locally
```bash
kubectl cp vault/consul-consul-server-0:/tmp/backup.snap ./vault-backup-$(date +%Y%m%d%H%M%S).snap
```

- copy to offsite storage (optional):
```bash
aws s3 cp vault-backup-$(date +%Y%m%d%H%M%S).snap s3://your-backup-bucket/vault-backups/
```

## restore snapshot

- Copy the snapshot to the pod: 
```bash
kubectl cp vault-backup-<timestamp>.snap vault/consul-consul-server-0:/tmp/backup.snap
```
- Restore:
```bash
kubectl exec -n vault consul-consul-server-0 -- consul snapshot restore /tmp/backup.snap
```

### vault config backup

- export policies
```bash
vault policy list > policies.txt
for policy in $(vault policy list); do
  vault policy read $policy > policy-$policy.hcl
done
```

- export auth methods
```bash
vault auth list > auth-methods.txt
vault read auth/kubernetes/config > kubernetes-auth-config.txt
vault list auth/kubernetes/role > kubernetes-roles.txt
for role in $(vault list -format=json auth/kubernetes/role | jq -r '.[]'); do
  vault read auth/kubernetes/role/$role > kubernetes-role-$role.txt
done
```
### backup kubernetes manifests

```bash
kubectl get all -n vault -o yaml > vault-k8s-resources.yaml
kubectl get pvc -n vault -o yaml >> vault-k8s-resources.yaml
kubectl get configmap -n vault -o yaml >> vault-k8s-resources.yaml
kubectl get secret -n vault -o yaml >> vault-k8s-resources.yaml
aws s3 cp vault-k8s-resources.yaml s3://your-backup-bucket/vault-config/
```
