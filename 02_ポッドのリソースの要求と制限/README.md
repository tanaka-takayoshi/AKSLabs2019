# ポッドのリソースの要求と制限

ポッドの要求(request)を設定した場合のスケジュールの動作を確認します。

まずここのサンプルを試すための名前空間`pod-resource`を作成します。

```
kubectl create namespace pod-resource
```

kubernetesのポッドにはリソースをどのくらい要求(request)もしくは制限(limit)するかの設定ができます。
既存のクラスターがどのくらいリソースを使っているかは`kubectl describe node`で確認できます。

```
...前略
Non-terminated Pods:         (11 in total)
  Namespace                  Name                                     CPU Requests  CPU Limits  Memory Requests  Memory Limits
  ---------                  ----                                     ------------  ----------  ---------------  -------------
  kube-system                coredns-79c89b8f4-kh5ct                  100m (5%)     0 (0%)      70Mi (1%)        170Mi (3%)
  kube-system                coredns-79c89b8f4-qsmtm                  100m (5%)     0 (0%)      70Mi (1%)        170Mi (3%)
  kube-system                coredns-autoscaler-6fcdb7d64-dth2p       20m (1%)      0 (0%)      10Mi (0%)        0 (0%)
  kube-system                heapster-7677c744b8-5gkr8                130m (6%)     130m (6%)   230Mi (5%)       230Mi (5%)
  kube-system                kube-proxy-qlwrq                         100m (5%)     0 (0%)      0 (0%)           0 (0%)
  kube-system                kube-svc-redirect-48pdl                  10m (0%)      0 (0%)      34Mi (0%)        0 (0%)
  kube-system                kubernetes-dashboard-6dffbcc8b9-6lhdz    100m (5%)     100m (5%)   50Mi (1%)        500Mi (10%)
  kube-system                metrics-server-7b97f9cd9-fbjdk           0 (0%)        0 (0%)      0 (0%)           0 (0%)
  kube-system                omsagent-6bg4c                           50m (2%)      150m (7%)   225Mi (4%)       300Mi (6%)
  kube-system                omsagent-rs-5754f7dfdb-4tnkl             50m (2%)      150m (7%)   100Mi (2%)       500Mi (10%)
  kube-system                tunnelfront-786685bdd6-g866f             10m (0%)      0 (0%)      64Mi (1%)        0 (0%)
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  CPU Requests  CPU Limits  Memory Requests  Memory Limits
  ------------  ----------  ---------------  -------------
  670m (34%)    530m (27%)  853Mi (18%)      1870Mi (40%)
Events:
  Type    Reason    Age   From                                  Message
  ----    ------    ----  ----                                  -------
  Normal  Starting  42m   kube-proxy, aks-nodepool1-34744700-0  Starting kube-proxy.
```

このクラスターはnodeが1つですが、現在 CPUの要求が34%、制限が27%、memoryの要求が18%、制限が40%使われています。
要求はPodが必要とするリソースの量であるため、利用可能な量を超える要求を持ったPodはノードにスケジュールできません。
ためにしに5000mメモリーを要求する`request1-pod.yaml`を配置してみると次のようなエラーがでます。

```
kubectl create -f request1-pod.yaml
```

```
kubectl get event -n pod-resource

LAST SEEN   FIRST SEEN   COUNT     NAME                      KIND      SUBOBJECT   TYPE      REASON             SOURCE              MESSAGE
39s         1m           4         mypod1.159bf7925e4053ee   Pod                   Warning   FailedScheduling   default-scheduler   0/1 nodes are available: 1 Insufficient cpu.
```

これは、作成されたPod定義が要求したメモリーの量を確保できるノードが見つからなかったためです。
kubernetesはノードごとに利用可能なリソースを検出します。
また、Podのスケジュール時には要求が考慮されますが、制限は考慮されません。

さて、このままでは少数のPodだけでノードのリソースを占有できてしまいます。
複数のチームで1つのクラスターを利用する場合、このような状況をふせぐために名前空間ごとに上限を設定できます。
`quota.yaml`のように上限を設定できます。

```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-app-team
spec:
  hard:
    cpu: "10"
    memory: 200Mi
    pods: "10"
```

次のコマンドで名前空間に適用します。

```
kubectl apply -f quota.yaml --namespace pod-resource
```

設定したQuotaの利用状況は`kubectl describe quota`で確認できます。

```
kubectl describe quota -n pod-resource

Name:       dev-app-team
Namespace:  pod-resource
Resource    Used  Hard
--------    ----  ----
cpu         0     10
memory      0     200Mi
pods        0     10
```

まずこの名前空間に要求のないPodを配置してみます。

```
kubectl create -f request2-pod.yaml
```

するとこのようなエラーメッセージでエラーが発生します。Quotaを設定した名前空間には、要求のないPodは配置できません。

```
Error from server (Forbidden): error when creating ".\\request2-pod.yaml": pods "mypod1" is forbidden: failed quota: dev-app-team: must specify cpu,memory
```

次に`request3-pod.yaml`を作成するとPodが作成されQuotaの利用状況が更新されます。

```
kubectl create -f request3-pod.yaml
```

```
kubectl describe quota -n pod-resource

Name:       dev-app-team
Namespace:  pod-resource
Resource    Used   Hard
--------    ----   ----
cpu         100m   10
memory      128Mi  200Mi
pods        1      10
```

さらにもう一つ同じPodを配置しようとすると、メモリーの要求が128Mi+128Mi=256Miとなり上限の200Miを超えるためエラーが発生することがわかります。

```
kubectl create -f request3-pod.yaml
```

```
Error from server (Forbidden): error when creating ".\\request3-pod.yaml": pods "mypod1" is forbidden: exceeded quota: dev-app-team, requested: memory=128Mi, used: memory=128Mi, limited: memory=200Mi
```

さて、名前空間ごとに上限を設定していると要求を定義していないPodを配置できませんが、limitrangeを使って名前空間ごとに既定値を設定できます。

```
apiVersion: v1
kind: LimitRange
metadata:
  name: my-limit-range
spec:
  limits:
  - default:
      memory: 48Mi
      cpu: 1000m
    defaultRequest:
      memory: 12Mi
      cpu: 100m
    type: Container
```

このように定義したlimitrangeをpod-resource名前空間に適用します。

```
kubectl apply -f limitrange.yaml -n pod-resource
```

定義したlimitrangeはdescribeコマンドで確認できます。

```
kubectl describe limitrange -n pod-resource
Name:       my-limit-range
Namespace:  pod-resource
Type        Resource  Min  Max  Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---  ---  ---------------  -------------  -----------------------
Container   cpu       -    -    100m             1              -
Container   memory    -    -    12Mi             48Mi           -
```

すると要求を持たないPodが配置できます。

```
kubectl create -f request2-pod.yaml
```

作成されたPodの定義を見ると既定値が適用されているのがわかります。

```
kubectl describe pod mypod2 -n pod-resource
```

```（一部抜粋）
Containers:
  mypod2:
    Container ID:   docker://12b5999c7c08e9ac62114d62050def7f7476a9e134ec1fa37c565234ad02905e
    Image:          nginx:1.15.5
    Image ID:       docker-pullable://nginx@sha256:b73f527d86e3461fd652f62cf47e7b375196063bbbd503e853af5be16597cb2e
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Mon, 06 May 2019 18:30:56 +0900
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     1
      memory:  48Mi
    Requests:
      cpu:     100m
      memory:  12Mi
```

## クリーンアップ

このドキュメントで利用したリソースを削除する場合は名前空間ごと削除します。

```
kubectl delete namespace pod-resource
```

## 参考資料

- [ポッドのリソースの要求と制限を定義する](https://docs.microsoft.com/ja-jp/azure/aks/developer-best-practices-resource-management#define-pod-resource-requests-and-limits)
- [リソース クォータを適用する](https://docs.microsoft.com/ja-jp/azure/aks/operator-best-practices-scheduler#enforce-resource-quotas)
- [Managing Compute Resources for Containers](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/)
- [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Configure Default Memory Requests and Limits for a Namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/)