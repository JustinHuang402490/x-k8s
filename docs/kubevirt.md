# Run Windows VM from an existing image
- Create two persistent volumes (PV) for CDI uploading
    `kubectl create -f pv-import-vm.yml`
    `kubectl create -f pv-import-vm-scratch.yml`
    
    - pv-import-vm.yml:
    ```yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: pv-import-vm
    spec:
      accessModes:
        - ReadWriteOnce
      volumeMode: Filesystem
      persistentVolumeReclaimPolicy: Recycle
      capacity:
        storage: 20Gi
      hostPath:
        path: /mnt/importVM/win10/pv
    ```
    
    - pv-import-vm-scratch.yml:
    ```yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: pv-import-vm-scratch
    spec:
      accessModes:
        - ReadWriteOnce
      volumeMode: Filesystem
      persistentVolumeReclaimPolicy: Recycle
      capacity:
        storage: 20Gi
      hostPath:
        path: /mnt/importVM/win10/pv-scratch
    ```
- See containerized data importer (CDI) upload proxy url
```kubectl get service -n cdi```

    ![](https://i.imgur.com/lyFh7cb.png)


- Uploading the VM image to the PVC
    ```
    ./virtctl image-upload dv pvc-win10vm \
    --size=20Gi \
    --insecure \
    --uploadproxy-url=https://10.233.31.233:443 \
    --image-path=/root/mydata/win10-sata-sparse.qcow2
    ```
    ![](https://i.imgur.com/Vye4gyN.png)
    - Note: 
        - You need to edit uploadproxy-url.
        - You would need to delete the data volume (DV) and upload the VM image again.
            - `kubectl delete dv pvc-win10vm`

- Create the Windows VM
    `kubectl create -f vm-win10.yml`
    - vm-win10.yml:
    ```yaml
    apiVersion: kubevirt.io/v1alpha3
    kind: VirtualMachine
    metadata:
      creationTimestamp: 2018-07-04T15:03:08Z
      generation: 1
      labels:
        kubevirt.io/os: win10
        special: key
      name: vm-win10
    spec:
      running: true
      template:
        metadata:
          creationTimestamp: null
          labels:
            kubevirt.io/domain: pvc-win10vm
            special: key
        spec:
          domain:
            cpu:
              cores: 1
            devices:
              blockMultiQueue: true
              disks:
              - disk:
                  bus: sata
                name: disk0
                cache: none
            machine:
              type: q35
            resources:
              requests:
                memory: 4096M
          volumes:
          - name: disk0
            persistentVolumeClaim:
              claimName: pvc-win10vm
    ```

- Check virtual machine instance (VMI) status
    `kubectl get vmi`
    
    ![](https://i.imgur.com/Cp6uXrf.png)
    - Note:
        - The status of VMI should be running.
        - Use the command to see details if there is any problem.
            - `kubectl describe vmi vm-win10`
    
- Connect to the VM
    - First way: using vnc directly.
        - `./virtctl vnc vm-win10`
    - Second way: using noVNC
        - `kubectl apply -f https://github.com/wavezhang/virtVNC/raw/master/k8s/virtvnc.yaml`
        - `kubectl get svc -n kubevirt virtvnc`
         
            ![](https://i.imgur.com/pKhuRUn.png)
            
            ![](https://i.imgur.com/nhnbx8S.png)

        > Reference:
        > https://kubevirt.io/2019/Access-Virtual-Machines-graphic-console-using-noVNC.html
---