# Rook

Rook은 Kubernetes cluster에서 동작하는 storage service를 위한 orchestrator이다.

Ceph는 설치를 하고 storage cluster를 유지하는데 많은 복잡한 점이 있다. Rook은 이런 어려움들을 자동화하고 간단하게 했다.

Rook은 operator로 구현이 된다. 즉, 관리자가 cluster의 desired state만 선언하면 되는 것이다. Rook operator는 요구된 설정을 감시하고 cluster에 configuration을 적용한다. cluster를 동작시키고, healthy하도록 유지시키는 모든 필요한 들을 포함하여 상태(state)가 operator에 의해서 관리된다.

