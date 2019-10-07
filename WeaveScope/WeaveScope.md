# WeaveScope

## 설치 방법

지원하는 Kubernetes version : v1.4 ~ v1.10

```bash
curl -o scope.yaml "https://cloud.weave.works/k8s/v1.10/scope.yaml"
```

`weave-scope-app.weave.svc.cluster.local:80`을 `weave-scope-app.weave.svc.<클러스터 이름>.local:80`으로 변경합니다.

```bash
sed -i 's/svc.cluster.local/svc.5gsamsung.local/g' scope.yaml
```



## 서비스 Expose

docs 상에서 서비스를 NodePort나 LoadBalancer로 expose하는 것은 추천하지 않습니다. weave scope를 통해서 누구든지 호스트와 컨테이너의 user interface control로 접속할 수 있기 때문입니다.

하지만 여기서는 NodePort를 통해서 expose하도록 하겠습니다.

`scope.yaml`을 다시 수정합니다.

`Service`에서 `spec.type= NodePort`를 추가하고 `spec.ports[].nodePort= 30010`등으로 원하는 포트를 넣어주세요.

혹은 공식 docs에 나온대로 `kubectl port-forward -n weave "$(kubectl get -n weave pod --selector=weave-scope-component=app -o jsonpath='{.items..metadata.name}')" 4040`을 통해서도 임시적으로 expose할 수 있습니다. 이때 namespace를 변경하였다면 weave scope의 namespace가 weave임에 주의해주세요.

## Kubectl Apply

마지막으로 `kubectl apply -f scope.yaml`을 통해서 배포하세요.

접속 url은 `<masterIP>:30010`입니다.