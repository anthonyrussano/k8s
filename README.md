## vault backup

### create snapshot

```bash
kubectl exec -n vault consul-consul-server-0 -- consul snapshot save /tmp/backup.snap
kubectl cp vault/consul-consul-server-0:/tmp/backup.snap ./vault-backup-$(date +%Y%m%d%H%M%S).snap
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