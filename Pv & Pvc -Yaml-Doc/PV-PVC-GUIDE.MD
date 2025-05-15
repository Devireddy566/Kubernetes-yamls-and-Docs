# Kubernetes PV and PVC Binding Guide for GCP

## 1. Persistent Volume (PV) Configuration
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pokemon-pv                                    # Name of your PV
  labels:
    topology.kubernetes.io/zone: us-central1-a        # Must match your GCP disk zone
    topology.kubernetes.io/region: us-central1        # Must match your GCP region
spec:
  capacity:
    storage: 10Gi                                     # Must match or exceed PVC request
  accessModes:
    - ReadWriteOnce                                  # Common for GCP disks
  persistentVolumeReclaimPolicy: Retain              # What happens when PVC is deleted
  storageClassName: standard                         # Must match PVC's storageClassName
  gcePersistentDisk:
    pdName: pokemon-disk                             # Must match your GCP disk name
    fsType: ext4                                     # Filesystem type
  nodeAffinity:                                      # Required for zonal disks
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: topology.kubernetes.io/zone
          operator: In
          values:
          - us-central1-a                            # Must match your GCP disk zone
        - key: kubernetes.io/arch
          operator: In
          values:
          - amd64                                    # Architecture of your nodes
```

## 2. Persistent Volume Claim (PVC) Configuration
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pokemon-pvc                                  # Name of your PVC
  namespace: dev                                     # Namespace where PVC will be used
spec:
  volumeName: pokemon-pv                            # Name of PV to bind to (optional but recommended)
  storageClassName: standard                        # Must match PV's storageClassName
  accessModes:
    - ReadWriteOnce                                # Must match PV's accessMode
  resources:
    requests:
      storage: 10Gi                                # Must not exceed PV's capacity
```

## 3. Key Points for Binding

### Storage Class Matching
- Both PV and PVC must have the same `storageClassName`
- Common classes in GCP: "standard", "premium-rwo"

### Capacity
- PVC requested storage must not exceed PV capacity
- Example: PV: 10Gi ≥ PVC: 10Gi

### Access Modes
- Must match between PV and PVC
- Common options:
  - `ReadWriteOnce (RWO)`: Single node read-write
  - `ReadOnlyMany (ROX)`: Multi-node read-only
  - `ReadWriteMany (RWX)`: Multi-node read-write (not supported by GCE disks)

### Zone and Region Labels
- PV must have correct topology labels for GCP:
  ```yaml
  topology.kubernetes.io/zone: us-central1-a
  topology.kubernetes.io/region: us-central1
  ```

### Node Affinity
- Required for zonal disks
- Must match your GCP zone
- Must include architecture if mixed architectures are used

## 4. Application Deployment Using PVC
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: your-app
  namespace: dev                    # Must match PVC namespace
spec:
  template:
    spec:
      volumes:
      - name: storage-volume
        persistentVolumeClaim:
          claimName: pokemon-pvc    # Must match PVC name
      containers:
      - name: app-container
        volumeMounts:
        - name: storage-volume
          mountPath: /data         # Mount path inside container
```

## 5. Binding Process
1. Create GCP disk in desired zone
2. Create PV referencing the disk
3. Create PVC in the desired namespace
4. Verify binding:
   ```bash
   kubectl get pv
   kubectl get pvc -n <namespace>
   ```
   Status should show "Bound" for both

## 6. Common Issues and Solutions

### 1. PV shows "Available"
- Check storageClassName matches
- Verify PVC exists in correct namespace
- Confirm capacity requirements match

### 2. PV shows "Bound" but disk shows "None" in GCP
- Normal until pod using PVC is running
- Deploy application using PVC
- Check pod status and events

### 3. Zone Mismatch
- Ensure PV zone matches GCP disk zone
- Update nodeAffinity in PV
- Update topology labels

### 4. Architecture Issues
- Add architecture in nodeAffinity
- Ensure nodes with matching architecture exist

## 7. Best Practices

### Deletion Order
Always delete resources in this order:
1. Delete Deployment/Pod using PVC
2. Delete PVC
3. Delete PV
4. Delete GCP disk (if needed)

### Labels and Annotations
- Use consistent labeling scheme
- Add meaningful labels for easier resource tracking
- Consider adding annotations for additional metadata

### Storage Planning
- Size PVs appropriately for expected growth
- Consider using storage classes for dynamic provisioning
- Plan for backup and disaster recovery

## 8. Quick Commands

### Resource Status
```bash
# Check PV status
kubectl get pv

# Check PVC status
kubectl get pvc -n <namespace>

# Check pod status
kubectl get pods -n <namespace>

# Get detailed information
kubectl describe pv <pv-name>
kubectl describe pvc -n <namespace> <pvc-name>
```

### Resource Deletion
```bash
# Delete in correct order
kubectl delete deployment <deployment-name> -n <namespace>
kubectl delete pvc <pvc-name> -n <namespace>
kubectl delete pv <pv-name>
```

Remember to replace placeholder values (like `pokemon-pv`, `us-central1-a`, etc.) with your actual resource names and zones.
