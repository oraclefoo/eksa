## EKS Anywhere bare metal GPU Operator
Steps and outputs are tested with the following:
* eksctl anywhere version v0.15.0
* ubuntu 20.04.5 LTS os image from image builder 1-25 channel
* 1 control node and 3 worker nodes
* GPU worker node - HPE DL360 Gen10 2xIntel(R) Xeon(R) Gold 6238R CPU @ 2.20GHz, 384GB memory, 2xNVIDIA Tesla T4
* EKS-A cluster v1.23.9-eks-68c1cba, containerd://1.5.9
* nvidia [gpu operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/overview.html )

### Install the nvidia driver on the gpu worker node
ssh to the gpu worker node, e.g. ssh ec2-user@10.10.80.56

1. Check the GPU is detected by the OS
```
sudo lspci | grep -i nvidia
```

		- Output
    ```
    12:00.0 3D controller: NVIDIA Corporation TU104GL [Tesla T4] (rev a1)
    d8:00.0 3D controller: NVIDIA Corporation TU104GL [Tesla T4] (rev a1)
    ```
2. Install ubuntu-drivers to detect the recommended nvidia driver
```
sudo apt-get update
sudo apt-get -y install ubuntu-drivers-common
sudo ubuntu-drivers devices
```

		- Output from ubuntu-drivers devices
    ```
    == /sys/devices/pci0000:11/0000:11:00.0/0000:12:00.0 ==
    modalias : pci:v000010DEd00001EB8sv000010DEsd000012A2bc03sc02i00
    vendor   : NVIDIA Corporation
		model    : TU104GL [Tesla T4]
		driver   : nvidia-driver-510 - distro non-free
		driver   : nvidia-driver-418-server - distro non-free
		driver   : nvidia-driver-450-server - distro non-free
		driver   : nvidia-driver-470-server - distro non-free
		driver   : nvidia-driver-525 - distro non-free
		driver   : nvidia-driver-515-server - distro non-free
		driver   : nvidia-driver-525-server - distro non-free
		driver   : nvidia-driver-530 - distro non-free recommended
		driver   : nvidia-driver-470 - distro non-free
		driver   : nvidia-driver-515 - distro non-free
		driver   : xserver-xorg-video-nouveau - distro free builtin
		```

3. Install the recommended driver. Note: autoinstall caused missing uuid issue where the server would not reboot
```
sudo apt -y install nvidia-driver-530
```
4. Reboot the node - sudo reboot

5. ssh to the gpu worker node and check the nvidia driver
```
sudo nvidia-smi
```

		- Output
		```
		+---------------------------------------------------------------------------------------+
		| NVIDIA-SMI 530.41.03              Driver Version: 530.41.03    CUDA Version: 12.1     |
		|-----------------------------------------+----------------------+----------------------+
		| GPU  Name                  Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
		| Fan  Temp  Perf            Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
		|                                         |                      |               MIG M. |
		===============|
		|   0  Tesla T4                        Off| 00000000:12:00.0 Off |                    0 |
		| N/A   54C    P0               28W /  70W|      2MiB / 15360MiB |      0%      Default |
		|                                         |                      |                  N/A |
		+-----------------------------------------+----------------------+----------------------+
		|   1  Tesla T4                        Off| 00000000:D8:00.0 Off |                    0 |
		| N/A   53C    P0               29W /  70W|      2MiB / 15360MiB |      7%      Default |
		|                                         |                      |                  N/A |
		| Processes:                                                                            |
		|  GPU   GI   CI        PID   Type   Process name                            GPU Memory |
		|        ID   ID                                                             Usage      |
		|  No running processes found                                                           
		```
### Install the gpu operator
On the admin node

6. Add nvidia repo to helm
```
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia && helm repo update
```

7. Install the gpu operator
```
helm install --wait --generate-name -n gpu-operator --create-namespace \
     nvidia/gpu-operator \
     --set driver.enabled=false
```
		- Output
		```
		NAME: gpu-operator-1682162417
		LAST DEPLOYED: Sat Apr 22 11:20:18 2023
		NAMESPACE: gpu-operator
		STATUS: deployed
		REVISION: 1
		TEST SUITE: None
		```
The gpu operator is in namespace gpu-opeartor. 

After a few minutes, check status 
```
kubectl get all -n gpu-operator
```

These pods are running on the nodes with the GPU
```
kubectl get po -o wide -n gpu-operator
```

		- Output
		```
		NAME                                                              READY   STATUS      RESTARTS   AGE     IP             NODE          NOMINATED NODE   READINESS GATES
		nvidia-container-toolkit-daemonset-rwcvn                          1/1     Running     0          4m52s   172.16.1.146   eksworker3    <none>           
		nvidia-cuda-validator-5wjhk                                       0/1     Completed   0          3m44s   172.16.1.20    eksworker3    <none>           
		nvidia-dcgm-exporter-j6r5f                                        1/1     Running     0          4m52s   172.16.1.65    eksworker3    <none>           
		nvidia-device-plugin-daemonset-9rftb                              1/1     Running     0          4m52s   172.16.1.52    eksworker3    <none>           
		nvidia-device-plugin-validator-9c2m6                              0/1     Completed   0          2m43s   172.16.1.17    eksworker3    <none>           
		nvidia-operator-validator-lcnzv                                   1/1     Running     0          4m52s   172.16.1.103   eksworker3    <none>         ` 
		```

Describe the node shows the gpu labels
```
kubectl describe no eksworker3
```

		- Output
		```
		Name:               eksworker3
		Roles:              <none>
		Labels:             beta.kubernetes.io/arch=amd64
												beta.kubernetes.io/os=linux
		….
												nvidia.com/cuda.driver.major=530
												nvidia.com/cuda.driver.minor=41
												nvidia.com/cuda.driver.rev=03
												nvidia.com/cuda.runtime.major=12
												nvidia.com/cuda.runtime.minor=1
												nvidia.com/gfd.timestamp=1682162569
												nvidia.com/gpu.compute.major=7
												nvidia.com/gpu.compute.minor=5
												nvidia.com/gpu.count=2
												nvidia.com/gpu.deploy.container-toolkit=true
												nvidia.com/gpu.deploy.dcgm=true
												nvidia.com/gpu.deploy.dcgm-exporter=true
												nvidia.com/gpu.deploy.device-plugin=true
												nvidia.com/gpu.deploy.driver=true
												nvidia.com/gpu.deploy.gpu-feature-discovery=true
												nvidia.com/gpu.deploy.node-status-exporter=true
												nvidia.com/gpu.deploy.operator-validator=true
												nvidia.com/gpu.family=turing
												nvidia.com/gpu.machine=ProLiant-DL360-Gen10
												nvidia.com/gpu.memory=15360
												nvidia.com/gpu.present=true
												nvidia.com/gpu.product=Tesla-T4
												nvidia.com/gpu.replicas=1
												nvidia.com/mig.capable=false
												nvidia.com/mig.strategy=single
		```

8. Create gpupod.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: cuda-vector-add
spec:
  restartPolicy: OnFailure
  containers:
  - name: cuda-vector-add
    image: "k8s.gcr.io/cuda-vector-add:v0.1"
    resources:
      limits:
        nvidia.com/gpu: 1
```

9. Apply gpupod.yaml
```kubectl apply -f gpupod.yaml```

		- Output
		```
		pod/cuda-vector-add created
		```

After a minute
```
kubectl get po
```

		- Output
		```
		NAME              READY   STATUS      RESTARTS   AGE
		cuda-vector-add   0/1     Completed   0          52s
		```

Check pod gpu limits and requests
```
kubectl describe po cuda-vector-add | grep gp
```

		- Output
		```
					nvidia.com/gpu:  1
					nvidia.com/gpu:  1
		```

10. Check the pod log
```
kubectl logs cuda-vector-add
```

		- Output
		```
		[Vector addition of 50000 elements]
		Copy input data from the host memory to the CUDA device
		CUDA kernel launch with 196 blocks of 256 threads
		Copy output data from the CUDA device to the host memory
		Test PASSED
		Done
		```
