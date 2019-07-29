#### docker image 로드할 때

```shell
for i in `ls *.tar`
do
docker load -i $i
done
```

#### docker image 저장할 때

```shell
docker save -o busybox.tar busybox
```

#### 특정 노드에 파드를 띄우고 싶을 때?

```yaml
oam_test.yaml
---
nodeSelector:
	kubernetes.io/hostname: worker1
```

