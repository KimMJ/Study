# Harbor

### harbor 요청정보

1. kubernetes cluster 이름
   cluster name : 5gpoc
   Master : k8s-master01(10.251.161.216)
   Node1 : k8s-node01(10.251.161.229)
   Node2 : k8s-node02(10.251.161.232)

2. master서버에서의 /etc/kubernetes/admin.conf

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5RENDQWJDZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRFNU1EY3hOakE0TVRNME1Wb1hEVEk1TURjeE16QTRNVE0wTVZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBSzZPCkxTL21mRU1uQjFFNDl4cm9HSC9CRDZQU0dIbXVqdWJlSmcweEMyTWFlc001QmlRY2EwTHJYaXdUR29OdnJscHIKdlVUcDBTVEYxZ2VQRENJMFlCVU9KL2o3cGNENzJLbHNWejRXT0ZCc1VtU0Fwc3UwK2d6RHZkNjBrcWJvR2pvWgpSZHlOUFl4WXAxQk1KZ3JXU0d4SGMwS09YVjg4UVV4VGZNK0Z3dWJ3TU42b0h2QUMvVXltQ21JWDFjL0R3NlowCkl0Unh2R2o1MEF0YlErbHNwNzNFUzNNYzE4U0c5d252NGRXS1BwOTliWU5iREJGL1dXR1lObHZvRml6Vm5BL0sKMWJ0Y3Q4NjREZi84NG0yc0NCNDdqK0taZ0EyRFR1NS8yZVhpTUxJSEVtZGlEVXdyaWtyZzZKc1NJbnpVYWUrdQpkREttekNHTDNVNmNrRVJHV1JVQ0F3RUFBYU1qTUNFd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFBVzJIV21pOTdNV3NZenBFNGh6Y0R3bi9NKzYKQzNicEl1Q3YyTGJlaDdkeEkyOXBDN2RlVFUzbjRhbDFubXRTeHBQUXg1c2EzL1dxemxrSWNJSTYyK0Y2QU1PSwpNQmNnTVNRaTdUMjllOEw0enZjVEV2ZWY1NHJUaFNsKzVYV3lUR2dBL01qNVQvYWtvcmpqWEZYaHNzMk1ycENECnkyVzhodytLdm5FcExVVUtOTHQ3TVljZHBzZ0dhWHgwRmtUL2ZScE1GNEZtVldSVUVwSWE2UEdxQjBXd3dlUHEKTGsreTgzTnhqa2JGRTdEb1hIRUVGUFhkZHdGR1hHVnYxaG9kRFRwbDh4U1BxcHhvZHcyL25EZVZCeFB5MWxTMgpKSStwZk05d1ZzNngvK0ZuYW96UmN6d2lKTnRSazliSHBiYk1VUlY4dkExL3VENzM3N2phanA3NW9jND0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
    server: https://10.251.161.216:6443
  name: 5gpoc
contexts:
- context:
    cluster: 5gpoc
    user: kubernetes-admin
  name: kubernetes-admin@5gpoc
current-context: kubernetes-admin@5gpoc
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM4akNDQWRxZ0F3SUJBZ0lJU2hxMnRPSDl3Vm93RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB4T1RBM01UWXdPREV6TkRGYUZ3MHlNREEzTVRVd09ERXpORFJhTURReApGekFWQmdOVkJBb1REbk41YzNSbGJUcHRZWE4wWlhKek1Sa3dGd1lEVlFRREV4QnJkV0psY201bGRHVnpMV0ZrCmJXbHVNSUlCSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQXdwMkh6Q3RFcmdwcWhPM3AKZkRvM2N0OEUzZU9sZ3B3NVRhaWNYbGRMVVJNVTJMTUgvbUtTY3BRSmphMEwyemU5NUFIc2xuWWlMV0lrRHhragoxMFlGK3ZLcmR1RW9kYzZmYlRDMnhBQzBFYXJteTltNUNTdVltTkM3cE9qcnlWa2VteG01UEFRQStFbVVqSERKCjR0VVY3akhxVkZHYXp0a3lvYXBxYndIUjVTdTdjRWhndFAwcnRCMHZWNnc0TXVLVlJQQkdJcEFIclJEQWRmQlYKcS9lKzUrd3lEVWdkdmtyaGp6ejkzUWl1bFFtejFXclZxYlFXR2NsSjM5dDJwbjd2MG5CQU94c091a3pwaHFEVwo5Y200RFV2WDZZLzRZRlRwRkdWajY0RHFXL0d2VTR1THNVRzRJQ2JyNjlkREkwQWJTYmRtTzFETEJwK3A4QytRCjFLQ2lxUUlEQVFBQm95Y3dKVEFPQmdOVkhROEJBZjhFQkFNQ0JhQXdFd1lEVlIwbEJBd3dDZ1lJS3dZQkJRVUgKQXdJd0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFBUlVydjJ1OWhKQy9xSWgyRlpJQVFYTTFBVWJWUUhzbzVxMgoyc1oxZ1h0dUlGRE5UazFHcjNqd3IrclBZaE5TWWdUd0xwN2RrOXlqL24wdnhaYnFoNEhvejFkL2RiRGlQY01RCkJGY3JjRXBBTG16RXh1dTdBczVBQ1htV0VRM3dWRTNINE1XRWdnRmQ1SG1qYXlYSlVVcm8vaUluVlRCbzVxWFIKbjVmcGM0OG8wZFhsbGdDYStLT1dCM2h4eVBhNUh3aXBXRHRrVklpSVRhK2VFbDJUajNIZkExajRabVlqT2JKTQo4OUZDNGFna0FrWGdDNis1VUN3TDQ1QmQyWWNxT3lkWFdhUmtpMHA2dzhOZ3d4R2haZ2FxRXZBaThhWm9td2V6CjZKU0FvY3JtVmNFcnZUL0o4Sm00cS9qd0tCcWVIOENOdnhCejZxQmsvaGdIdzVodVRNVT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFb3dJQkFBS0NBUUVBd3AySHpDdEVyZ3BxaE8zcGZEbzNjdDhFM2VPbGdwdzVUYWljWGxkTFVSTVUyTE1ICi9tS1NjcFFKamEwTDJ6ZTk1QUhzbG5ZaUxXSWtEeGtqMTBZRit2S3JkdUVvZGM2ZmJUQzJ4QUMwRWFybXk5bTUKQ1N1WW1OQzdwT2pyeVZrZW14bTVQQVFBK0VtVWpIREo0dFVWN2pIcVZGR2F6dGt5b2FwcWJ3SFI1U3U3Y0VoZwp0UDBydEIwdlY2dzRNdUtWUlBCR0lwQUhyUkRBZGZCVnEvZSs1K3d5RFVnZHZrcmhqeno5M1FpdWxRbXoxV3JWCnFiUVdHY2xKMzl0MnBuN3YwbkJBT3hzT3VrenBocURXOWNtNERVdlg2WS80WUZUcEZHVmo2NERxVy9HdlU0dUwKc1VHNElDYnI2OWRESTBBYlNiZG1PMURMQnArcDhDK1ExS0NpcVFJREFRQUJBb0lCQUNQajFVVkx2WEN6THcydwpxbHhraUJGQkc2Nm42ci81ZTMvYzFtbDNnOFpCMUpoWisrRm40RVlORXUzenVib0Z2NWtxMmF5dHdJUEtFNGhOClJKVFFydzJtYndTUFpWekViQlpBNDVPbDVZOVVpeGVRNFZUVm0yQ2pMZGV0dEwzL0YydlhCSmdTelBMODdzNHYKaHF1MFRFVVBJMzNGUnQxYXBNRzNvY1V5K3JoZVU3bFdpUUdMQjZ0KzJhQmdrd0Q0TjhNYndzWjQvZjdVWG9PbQozaGVQNlJCSXJkTU1LQ0ZZOUNUb0l5bGFjZnN6ZHFKL2pmZklIQnJlSFZkUVVIdUVFbmZscURvZ1lOUDJ2OUxxCjJ0NVBRclorbnVGUFFhVEJlT1dWRm5JVzdkQnhUaHhON0p1ZVZjS0NrekpJMEtRdHg1dkdPc2NkdktMRjV6aFEKc2lZNkpMa0NnWUVBOEE1VG15elpXeVdhYXcvVms2WnhwUWVRcHk1V29zalJWeXduWklET3lGUFlZR2xmMnIwZApRUGF4QnVmKzRsMGZUQmNGLzU0QWxBN3lVUE9rbFNlYnQ0SEw1UFlqdWNsdnZOQmlQVXpCNGdIUHowTmpxZEYzCkFQbkZZeHU4Q0JRMGU0dkNlOVFEaUgxVCtiVmFpa0Q4SEhKNFM4S1NpZEdWdThEWkZGOWVSb2NDZ1lFQXo0cVQKcXcyZ0NqT2hpbml0eG9pUGJLSEl1ZVJYelFZV2kzbUtpL1JCNXR1TW90VzZCTWlrbC9UdlF5TUNzZUJIT2VoZgpKa3VnRzZ3blZLYWVnVlhJbmNaemQ0ZTNycWlaL2NmYWFvY2pMbFd0SlZTVU5BbmRrenA3ZkZFK2JQS21lK1AvCm16Uyt1cEdvN1o2U3NHUGpQaVFEQlZpcDEwbnBrSzljWFdMbzZVOENnWUJuY3FVUXorank0R2VGRDVQSVJ3ZmUKU0Q1TDdTb2tpRW0rT1NiWXByRjFucncxLy9Md3ZtSm01bWd2UTdhUk1mUVV4Qzh2a3BWSk9JK3YxdTdyMzkrNAoydFJVM01WVWdMd0lML3pGMGRnVFh4aUFodGZpRElRdUJYVE1XdDFTMWZJdjgzQmlFR0ZkWmpUVC9SVUJVelBSCnhucVVtMHF1M1lTYkhtWHQ0NU1xN1FLQmdCd2RXNis2WXNtL0FNMHZWK3NqS0xyQWw5Nkd6bFlaMHdnRjZQelkKayt6Z0pRY1NDT2NJL3pNT25UTHRGVHBmZFlha3NlOFFJNXBjRWQvbnltVWU1OVJudzlDWGRBeVhEblZRazRnRwowbjgrWC94RW51Y0Z4eHhndWNXM2c4dGllNmNnMWNtQ3RhdTBlN3ZrMVY1THljYnJQZldGYzB5VTJLMGU5Rlk2ClJlOEZBb0dCQUlzUFFXaG1yMFRPTHpSK203dFpVMmtEYUNySFkyaWhaVjhRVnlBbEk5c0pFQ0VxdzJCRjN1bmgKb1pRa2xSRnlzNDl2dkxvL1dRWTNLbWhvbzJ0b2NneXlyZ0lYTU04eTFtY2Z0VzBTZUdpVFJNaW5ZUTJhcm1kUQpLaXFlMEl0QXRsWm95M0UydXljVTZnMVN1b2dSbm5qNWhDTy9IMjUwUmhwMW9qck00VDB2Ci0tLS0tRU5EIFJTQSBQUklWQVRFIEtFWS0tLS0tCg==
```

3. yaml

* nrf-oamp-test-to-onlyPV_cicd.yaml

  ```yaml
  # Copyright 2017 Istio Authors
  #
  #   Licensed under the Apache License, Version 2.0 (the "License");
  #   you may not use this file except in compliance with the License.
  #   You may obtain a copy of the License at
  #
  #       http://www.apache.org/licenses/LICENSE-2.0
  #
  #   Unless required by applicable law or agreed to in writing, software
  #   distributed under the License is distributed on an "AS IS" BASIS,
  #   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  #   See the License for the specific language governing permissions and
  #   limitations under the License.
  
  ##################################################################################################
  # Ratings service
  ##################################################################################################
  
  # Persistent Volumes for PLD
  # pld /storage folder
  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: pld-storage-pvc
    namespace: nrf-cicd
  spec:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 100Mi
  ---
  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: pld-package-pvc
    namespace: nrf-cicd
  spec:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 100Mi
  ---
  # End of configuration for PLD Persistent Volumes
  
  apiVersion: v1
  kind: Service
  metadata:
    name: oam
    namespace: nrf-cicd
    labels:
      app: oamp
      service: oam
  spec:
    ports:
    - port: 9839
      name: http2pm
    - port: 17680
      name: tcpinternalpm1
    - port: 17683
      name: tcpinternalpm2
    - port: 17416
      name: tcpinternalui1
    - port: 17417
      name: tcpinternalui2
    - port: 17418
      name: tcpinternalui3
    - port: 17419
      name: tcpinternalui4
    - port: 17608
      name: tcpinternalssmserver
    - port: 23892
      name: tcpinternalncdh0
    - port: 23893
      name: tcpinternalncdh1
    - port: 23894
      name: tcpinternalncdh2
    - port: 23895
      name: tcpinternalncdh3
    - port: 23896
      name: tcpinternalncdh4
    - port: 23897
      name: tcpinternalncdh5
    - port: 23898
      name: tcpinternalncdh6
    - port: 23899
      name: tcpinternalncdh7
    - port: 22
      name: sftpserver
    selector:
      app: oamp
  
  ---
  
  apiVersion: extensions/v1beta1
  kind: Deployment
  metadata:
    name: oamp
    namespace: nrf-cicd
    labels:
      app: oamp
  #    version: v1
  spec:
    replicas: 1
    template:
      metadata:
        labels:
          app: oamp
        #Commented out, as it was throwing issue.
        #annotations:
          #container.apparmor.security.beta.kubernetes.io/sftp: runtime/default
  #        version: v1
      spec:
        hostAliases:                # to be deleted in pkg.
        - ip: "127.0.0.1"
          hostnames:
          - "iiip0"
          - "oiip0"
          - "diip0"
    # These containers are run during pod initialization
        containers:
        - name: pm
          image: oampmserver
          imagePullPolicy: IfNotPresent
          ports:
          - containerPort: 9839
          - containerPort: 17680
          - containerPort: 17683
          volumeMounts:
            - name: host-vol-1
              mountPath: /storage
            - name: host-vol-2
              mountPath: /package
  
        - name: ui
          image: nwharbor.sec.samsung.net/core_svr19a1/nrf-ui
          #command:
          #- /hello/run_oam.sh
          imagePullPolicy: Always
          ports:
          - containerPort: 17416
          - containerPort: 17417
          - containerPort: 17418
          - containerPort: 17419
          volumeMounts:
            - name: host-vol-1
              mountPath: /storage
            - name: host-vol-2
              mountPath: /package
  
        - name: ssmserver
          image: oamssmserver
          #command:
          #- /hello/run_oam.sh
          imagePullPolicy: IfNotPresent
          ports:
          - containerPort: 17608
          volumeMounts:
            - name: host-vol-1
              mountPath: /storage
            - name: host-vol-2
              mountPath: /package
  
        - name: ncdh 
          image: ncdh_demo.image
          imagePullPolicy: IfNotPresent
          ports:
          - containerPort: 23892
          - containerPort: 23893
          - containerPort: 23894
          - containerPort: 23895
          - containerPort: 23896
          - containerPort: 23897
          - containerPort: 23898
          - containerPort: 23899
          volumeMounts:
            - name: host-vol-1
              mountPath: /storage
            - name: host-vol-2
              mountPath: /package
  
        - name: sftpserver 
          image: atmoz/sftp:withchanges
          imagePullPolicy: IfNotPresent
          env:
          - name: PASSWORD
            valueFrom:
              secretKeyRef:
                name: sftp-server-sec
                key: password
          args: ["user:$(PASSWORD):1001:1001:incoming,outgoing"] #create users and dirs
          ports:
            - containerPort: 22
          volumeMounts:
            - mountPath: /home/user/.ssh/keys
              name: sftp-public-keys
              readOnly: true
            - name: host-vol-1
              mountPath: /storage
            - name: host-vol-2
              mountPath: /package
  
          securityContext:
            capabilities:
              add: ["SYS_ADMIN"]
  
        volumes:
          - name: host-vol-1 # storage
            persistentVolumeClaim:
              claimName: pld-storage-pvc
          - name: host-vol-2 # package
            persistentVolumeClaim:
              claimName: pld-package-pvc
          - name: sftp-public-keys
            configMap:
              name: sftp-public-keys
  ---
  
  ```

  