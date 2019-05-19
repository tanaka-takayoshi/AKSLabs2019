# テイントと容認

テイントと容認は、次に説明するnodeSelector同様、Podを特定のnodeに配置するための機能です。
nodeSelectorと異なる点は、テイントと容認は「Podに容認が設定されていないと、テイントされたnodeに配置されない」という点です。

## ノードの追加

このドキュメントのサンプルを実行するためにノードを3台に増やします。まず、ノードプールの名前を確認するために以下のコマンドを実行します。
必要に応じて、リソースグループの指定(`-g`オプション)、クラスター名の置き換えをしてください。

```
az aks show -n decode2019cluster  --query agentPoolProfiles
```

次のように出力された場合、`count`が1なので3台に増やします。`name`がnodepool1なのでこれを使います。

```
[
  {
    "count": 1,
    "maxPods": 110,
    "name": "nodepool1",
    "osDiskSizeGb": 100,
    "osType": "Linux",
    "storageProfile": "ManagedDisks",
    "vmSize": "Standard_DS2_v2"
  }
]
```

次のコマンドでノードを3台に増やします。

```
az aks scale -n decode2019cluster --node-count 3 --nodepool-name nodepool1
```

しばらく待ったあとにJSONが出力されますが、`agentPoolProfiles`の`count`が3であれば成功しています。

## テイントの設定

まずノードの名前を確認します。

```
kubectl get nodes
```

```
NAME                       STATUS    ROLES     AGE       VERSION
aks-nodepool1-34744700-0   Ready     agent     5d        v1.12.7
aks-nodepool1-34744700-1   Ready     agent     35m       v1.12.7
aks-nodepool1-34744700-2   Ready     agent     35m       v1.12.7
```
`aks-nodepool1-34744700-1`と`aks-nodepool1-34744700-2`にそれぞれ異なるテイントを設定します。

```
kubectl taint nodes aks-nodepool1-34744700-1 key1=value1:NoSchedule
```

```
kubectl taint nodes aks-nodepool1-34744700-2 key2=value2:NoSchedule key3=value3:PreferNoSchedule
```

出力結果は省略しますが、テイントの状態も`kubectl describe node`で確認できます。

まず、容認のないpodを配置してみます。

```
kubectl create namespace taint-test
kubectl create -f notoleration-pod.yaml -n taint-test
```

このdeploymentはreplica3つなので、複数のノードの3つのPodが配置されることが期待されますが、何度デプロイしても`aks-nodepool1-34744700-0`にしかスケジュールされません。

```
kubectl get pods -n taint-test -o wide
```

```
NAME                                 READY     STATUS    RESTARTS   AGE       IP            NODE
mypod1-deployment-5dfbc5d9fb-5tdjt   1/1       Running   0          11s       10.244.0.15   aks-nodepool1-34744700-0
mypod1-deployment-5dfbc5d9fb-kfqsb   1/1       Running   0          11s       10.244.0.16   aks-nodepool1-34744700-0
mypod1-deployment-5dfbc5d9fb-rnv74   1/1       Running   0          11s       10.244.0.17   aks-nodepool1-34744700-0
```

次にkey1=value1という容認を設定したPodを配置してみます。このように設定します。

```
spec:
  containers:
  - image: nginx:1.15.5
    name: mypod2
  tolerations:
  - key: "key1"
    operator: "Equal"
    value: "value1"
    effect: "NoSchedule"
```

```
kubectl create -n taint-test -f key1toleration-pod.yaml
```

容認を設定しているので`aks-nodepool1-34744700-1`およびテイントが設定されていない`aks-nodepool1-34744700-0`にスケジュールされましたが、`aks-nodepool1-34744700-2`は別のテイントが設定されているためスケジュールされていません。
replica数8つで配置が偏っているのは、もともと`aks-nodepool1-34744700-0`に配置されているPodが多いためです。

```
kubectl get pods -n taint-test -o wide
NAME                                 READY     STATUS    RESTARTS   AGE       IP            NODE
mypod1-deployment-5dfbc5d9fb-5tdjt   1/1       Running   0          5m        10.244.0.15   aks-nodepool1-34744700-0
mypod1-deployment-5dfbc5d9fb-kfqsb   1/1       Running   0          5m        10.244.0.16   aks-nodepool1-34744700-0
mypod1-deployment-5dfbc5d9fb-rnv74   1/1       Running   0          5m        10.244.0.17   aks-nodepool1-34744700-0
mypod2-deployment-54b684599b-24mtz   1/1       Running   0          43s       10.244.1.6    aks-nodepool1-34744700-1
mypod2-deployment-54b684599b-7hxt8   1/1       Running   0          43s       10.244.1.8    aks-nodepool1-34744700-1
mypod2-deployment-54b684599b-97rpm   1/1       Running   0          43s       10.244.1.12   aks-nodepool1-34744700-1
mypod2-deployment-54b684599b-9ps8s   1/1       Running   0          43s       10.244.1.10   aks-nodepool1-34744700-1
mypod2-deployment-54b684599b-f9fqg   1/1       Running   0          43s       10.244.1.9    aks-nodepool1-34744700-1
mypod2-deployment-54b684599b-jpqxc   1/1       Running   0          43s       10.244.0.18   aks-nodepool1-34744700-0
mypod2-deployment-54b684599b-pq8zc   1/1       Running   0          43s       10.244.1.7    aks-nodepool1-34744700-1
mypod2-deployment-54b684599b-svhr4   1/1       Running   0          43s       10.244.1.11   aks-nodepool1-34744700-1
```

さらに、key2:value2を設定したPodを配置してみます。

```
kubectl create -n taint-test -f key1toleration-pod.yaml
```

`kubectl taint nodes aks-nodepool1-34744700-2 key2=value2:NoSchedule key3=value3:PreferNoSchedule`と`aks-nodepool1-34744700-2`にはkey2とkey3という2つのテイントが設定されています。
しかし。key3のテイントは`PreferNoSchedule`であるため、配置ができない場合は対応する容認がなくてもスケジュール可能です。
その代わりkey1の容認を持っていないため、`aks-nodepool1-34744700-1`にはスケジュールされません。

```
kubectl get pods -n taint-test -o wide
```

```
NAME                                 READY     STATUS    RESTARTS   AGE       IP            NODE
mypod1-deployment-5dfbc5d9fb-5tdjt   1/1       Running   0          18m       10.244.0.15   aks-nodepool1-34744700-0
mypod1-deployment-5dfbc5d9fb-kfqsb   1/1       Running   0          18m       10.244.0.16   aks-nodepool1-34744700-0
mypod1-deployment-5dfbc5d9fb-rnv74   1/1       Running   0          18m       10.244.0.17   aks-nodepool1-34744700-0
mypod2-deployment-54b684599b-24mtz   1/1       Running   0          13m       10.244.1.6    aks-nodepool1-34744700-1
mypod2-deployment-54b684599b-7hxt8   1/1       Running   0          13m       10.244.1.8    aks-nodepool1-34744700-1
mypod2-deployment-54b684599b-97rpm   1/1       Running   0          13m       10.244.1.12   aks-nodepool1-34744700-1
mypod2-deployment-54b684599b-9ps8s   1/1       Running   0          13m       10.244.1.10   aks-nodepool1-34744700-1
mypod2-deployment-54b684599b-f9fqg   1/1       Running   0          13m       10.244.1.9    aks-nodepool1-34744700-1
mypod2-deployment-54b684599b-jpqxc   1/1       Running   0          13m       10.244.0.18   aks-nodepool1-34744700-0
mypod2-deployment-54b684599b-pq8zc   1/1       Running   0          13m       10.244.1.7    aks-nodepool1-34744700-1
mypod2-deployment-54b684599b-svhr4   1/1       Running   0          13m       10.244.1.11   aks-nodepool1-34744700-1
mypod3-deployment-76d4d98997-bh76p   1/1       Running   0          4m        10.244.0.22   aks-nodepool1-34744700-0
mypod3-deployment-76d4d98997-cg8wx   1/1       Running   0          4m        10.244.0.19   aks-nodepool1-34744700-0
mypod3-deployment-76d4d98997-fq6ll   1/1       Running   0          4m        10.244.2.4    aks-nodepool1-34744700-2
mypod3-deployment-76d4d98997-kxdgs   1/1       Running   0          4m        10.244.0.23   aks-nodepool1-34744700-0
mypod3-deployment-76d4d98997-lh8nz   1/1       Running   0          4m        10.244.0.21   aks-nodepool1-34744700-0
mypod3-deployment-76d4d98997-n24v5   1/1       Running   0          4m        10.244.2.3    aks-nodepool1-34744700-2
mypod3-deployment-76d4d98997-t49nv   1/1       Running   0          4m        10.244.2.5    aks-nodepool1-34744700-2
mypod3-deployment-76d4d98997-vbjd8   1/1       Running   0          4m        10.244.0.20   aks-nodepool1-34744700-0
```

`PreferNoSchedule`と`NoSchedule`はスケジュールの際に考慮されますが、`NoExecute`は設定した時点で実行されていたPodも別のNodeに移動させられます。

```
kubectl taint node aks-nodepool1-34744700-1 key1=value3:NoExecute
```

key1の値をvalue3に変えたため、key1:value1という容認を設定していたPodは別のNodeに移動しました。

```
kubectl get pods -n taint-test -o wide
```

```
NAME                                 READY     STATUS    RESTARTS   AGE       IP            NODE
mypod1-deployment-5dfbc5d9fb-5tdjt   1/1       Running   0          32m       10.244.0.15   aks-nodepool1-34744700-0
mypod1-deployment-5dfbc5d9fb-kfqsb   1/1       Running   0          32m       10.244.0.16   aks-nodepool1-34744700-0
mypod1-deployment-5dfbc5d9fb-rnv74   1/1       Running   0          32m       10.244.0.17   aks-nodepool1-34744700-0
mypod2-deployment-54b684599b-5tpcp   1/1       Running   0          1m        10.244.0.29   aks-nodepool1-34744700-0
mypod2-deployment-54b684599b-6xkqh   1/1       Running   0          1m        10.244.0.30   aks-nodepool1-34744700-0
mypod2-deployment-54b684599b-87wks   1/1       Running   0          1m        10.244.0.24   aks-nodepool1-34744700-0
mypod2-deployment-54b684599b-bhrdj   1/1       Running   0          1m        10.244.0.26   aks-nodepool1-34744700-0
mypod2-deployment-54b684599b-jpqxc   1/1       Running   0          27m       10.244.0.18   aks-nodepool1-34744700-0
mypod2-deployment-54b684599b-mfmbs   1/1       Running   0          1m        10.244.0.25   aks-nodepool1-34744700-0
mypod2-deployment-54b684599b-vpqkm   1/1       Running   0          1m        10.244.0.27   aks-nodepool1-34744700-0
mypod2-deployment-54b684599b-wgzxc   1/1       Running   0          1m        10.244.0.28   aks-nodepool1-34744700-0
mypod3-deployment-76d4d98997-bh76p   1/1       Running   0          18m       10.244.0.22   aks-nodepool1-34744700-0
mypod3-deployment-76d4d98997-cg8wx   1/1       Running   0          18m       10.244.0.19   aks-nodepool1-34744700-0
mypod3-deployment-76d4d98997-fq6ll   1/1       Running   0          18m       10.244.2.4    aks-nodepool1-34744700-2
mypod3-deployment-76d4d98997-kxdgs   1/1       Running   0          18m       10.244.0.23   aks-nodepool1-34744700-0
mypod3-deployment-76d4d98997-lh8nz   1/1       Running   0          18m       10.244.0.21   aks-nodepool1-34744700-0
mypod3-deployment-76d4d98997-n24v5   1/1       Running   0          18m       10.244.2.3    aks-nodepool1-34744700-2
mypod3-deployment-76d4d98997-t49nv   1/1       Running   0          18m       10.244.2.5    aks-nodepool1-34744700-2
mypod3-deployment-76d4d98997-vbjd8   1/1       Running   0          18m       10.244.0.20   aks-nodepool1-34744700-0
```

## クリーンアップ

このドキュメントで利用したリソースを削除する場合は名前空間ごと削除します。
加えて、ノードに設定したテイントを削除します。

```
kubectl delete namespace taint-test
kubectl taint node aks-nodepool1-34744700-1 key1-
kubectl taint node aks-nodepool1-34744700-2 key2- key3-
```

## 参考資料

- [テイントと容認を使用して専用のノードを提供する](https://docs.microsoft.com/ja-jp/azure/aks/operator-best-practices-advanced-scheduler#provide-dedicated-nodes-using-taints-and-tolerations)
- [Taints and Tolerations](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/)