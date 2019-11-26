# Certificates

client certificate 인증을 사용할 때 `easyrsa`, `openssl`, `cfssl`을 통해서 수동으로 certificates를 생성할 수 있다.

### easyrsa

easyrsa는 클러스터에 대한 certificate를 수동으로 생성하는 것이다.

1. easyrsa3을 다운로드하고 설치하라.

   ```
   curl -LO https://storage.googleapis.com/kubernetes-release/easy-rsa/easy-rsa.tar.gz
   tar xzf easy-rsa.tar.gz
   cd easy-rsa-master/easyrsa3
   ./easyrsa init-pki
   ```

2. 새로운 certificate authority (CA)를 생성하라. `--batch`는 자동모드를 설정한다; `--req-cn`은 CA의 새로운 root certificate에 대한 Common Name (CN)을 지정한다.
   ``./easyrsa --batch "--req-cn=${MASTER_IP}@`date +%s`" build-ca nopass``

3. server certificate와 key를 생성한다. `--subject-alt-name` argument는 API server가 access할 수 있는 사용 가능한 IPs와 DNS names를 설정한다. `MASTER_CLUSTER_IP`는 보통 API server와 controller manager component에 대해 `--service-cluster-ip-range` argument로 지정한 service CIDR의 첫 IP이다. `--days` argument는 certificate가 만료될 날짜를 선택하는 것이다. 아래의 sample은 또한 `cluster.local`을 default DNS domain name으로 사용한다고 가정한 것이다.

   ```
   ./easyrsa --subject-alt-name="IP:${MASTER_IP},"\
   "IP:${MASTER_CLUSTER_IP},"\
   "DNS:kubernetes,"\
   "DNS:kubernetes.default,"\
   "DNS:kubernetes.default.svc,"\
   "DNS:kubernetes.default.svc.cluster,"\
   "DNS:kubernetes.default.svc.cluster.local" \
   --days=10000 \
   build-server-full server nopass
   ```

4. `pki/ca.crt`, `pki/issued/server.crt`, `pki/private/server.key`를 directory로 복사한다.

5. 다음의 parameter를 API server start parameter로 추가하라.

   ```
   --client-ca-file=/yourdirectory/ca.crt
   --tls-cert-file=/yourdirectory/server.crt
   --tls-private-key-file=/yourdirectory/server.key
   ```

### openssl

openssl은 cluster에 대해 수동으로 certificate를 생성할 수 있다.

1. ca.key를 2048bit로 생성한다.

   ```bash
   openssl genrsa -out ca.key 2048 
   ```

2. ca.key에 따라 ca.crt를 생성한다. (`-days`를 사용하여 certificate의 유효기간을 설정한다.)

   ```bash
   openssl req -x509 -new -nodes -key ca.key -subj "/CN=${MASTER_IP}" -days 10000 -out ca.crt
   ```

3. server.key를 2048bit로 생성한다.

   ```bash
   openssl genrsa -out server.key 2048
   ```

4. 생성한 Certificate Signing Request (CSR)에 대한 config file을 생성한다. 이 파일(e.g. `csr.conf`)을 저장하기 전에 <>로 마킹이 된 곳(e.g. `<MASTER_IP>`)의 값을 실제 값으로 바꾸어라. `MASTER_CLUSTER_IP`의 값이 이전 subsection에서 설명한 API server의 service cluster IP이다. 아래의 샘플은 `cluster.local`이 default DNS domain name이라고 가정한다.

   ```
   [ req ]
   default_bits = 2048
   prompt = no
   default_md = sha256
   req_extensions = req_ext
   distinguished_name = dn
   
   [ dn ]
   C = <country>
   ST = <state>
   L = <city>
   O = <organization>
   OU = <organization unit>
   CN = <MASTER_IP>
   
   [ req_ext ]
   subjectAltName = @alt_names
   
   [ alt_names ]
   DNS.1 = kubernetes
   DNS.2 = kubernetes.default
   DNS.3 = kubernetes.default.svc
   DNS.4 = kubernetes.default.svc.cluster
   DNS.5 = kubernetes.default.svc.cluster.local
   IP.1 = <MASTER_IP>
   IP.2 = <MASTER_CLUSTER_IP>
   
   [ v3_ext ]
   authorityKeyIdentifier=keyid,issuer:always
   basicConstraints=CA:FALSE
   keyUsage=keyEncipherment,dataEncipherment
   extendedKeyUsage=serverAuth,clientAuth
   subjectAltName=@alt_names
   ```

5. config file을 기반으로 certificate signing request를 생성한다.

   ```bash
   openssl req -new -key server.key -out server.csr -config csr.conf
   ```

6. ca.key, ca.crt, server.csr을 사용하여 server certificate를 생성한다.

   ```bash
   openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key \
   -CAcreateserial -out server.crt -days 10000 \
   -extensions v3_ext -extfile csr.conf
   ```

7. certificate를 본다.

   ```bash
   openssl x509 -noout -text -in ./server.crt
   ```

마지막으로 같은 파라미터를 API server start parameters로 추가한다.

### cfssl

cfssl은 certificate generation의 또다른 툴이다.

1. 아래에 보이는 것처럼 command line tools을 다운로드하고 설치한다. hardware architecture와 사용할 cfssl 버전에 따라서 아래의 샘플 커맨드를 적용시켜야 한다.

   ```bash
   curl -L https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -o cfssl
   chmod +x cfssl
   curl -L https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -o cfssljson
   chmod +x cfssljson
   curl -L https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -o cfssl-certinfo
   chmod +x cfssl-certinfo
   ```

2. artifacts를 유지하고 cfssl을 초기화하기 위해 디렉토리를 생선한다.

   ```bash
   mkdir cert
   cd cert
   ../cfssl print-defaults config > config.json
   ../cfssl print-defaults csr > csr.json
   ```

3. CA file을 생성하기 위한 JSON config file을 생성한다. 다음은 `ca-config.json` 예시이다.

   ```json
   {
     "signing": {
       "default": {
         "expiry": "8760h"
       },
       "profiles": {
         "kubernetes": {
           "usages": [
             "signing",
             "key encipherment",
             "server auth",
             "client auth"
           ],
           "expiry": "8760h"
         }
       }
     }
   }
   ```

4. CA certificate signing request (CSR)을 위한 JSON config file을 생성한다. 다음은 `ca-csr.json` 파일이다. <>로 마킹된 값들을 원하는 실제값으로 바꾸어야한다.

   ```json
   {
     "CN": "kubernetes",
     "key": {
       "algo": "rsa",
       "size": 2048
     },
     "names":[{
       "C": "<country>",
       "ST": "<state>",
       "L": "<city>",
       "O": "<organization>",
       "OU": "<organization unit>"
     }]
   }
   ```

5. CA key (`ca-key.pem`)과 certificate (`ca.pem`)을 생성한다.
   ```bash
   ../cfssl gencert -initca ca-csr.json | ../cfssljson -bare ca
   ```

6. API server에 대한 key와 certificate를 생성하기 위한 JSON file을 작성한다. 다음은 `server-csr.json` 파일이다. <>로 마킹된 값들을 실제 사용할 값으로 변경해야한다. `MASTER_CLUSTER_IP`는 이전의 subsection에 설명된 API server에 대한 service cluster IP이다. 아래의 sample은 `cluster.local`이 default DNS domain name이라고 가정한다.

   ```json
   {
     "CN": "kubernetes",
     "hosts": [
       "127.0.0.1",
       "<MASTER_IP>",
       "<MASTER_CLUSTER_IP>",
       "kubernetes",
       "kubernetes.default",
       "kubernetes.default.svc",
       "kubernetes.default.svc.cluster",
       "kubernetes.default.svc.cluster.local"
     ],
     "key": {
       "algo": "rsa",
       "size": 2048
     },
     "names": [{
       "C": "<country>",
       "ST": "<state>",
       "L": "<city>",
       "O": "<organization>",
       "OU": "<organization unit>"
     }]
   }
   ```

7. API server에 대한 key와 certificate를 생성한다. default로 `server-key.pem`과 `server.pem`에 저장된다.

   ```bash
   ../cfssl gencert -ca=ca.pem -ca-key=ca-key.pem \
   --config=ca-config.json -profile=kubernetes \
   server-csr.json | ../cfssljson -bare serverD
   ```

## Distributing Self-Signed CA Certificate

client node는 self-signed CA certificate를 유효하다고 인식하지 않을 것이다. non-production deployment에 대해서 또는 기업 방화벽이 동작하는 deployment에 대해서 self-signed CA certificate를 모든 클라이언트에 대해서 분배할 수 있고 유효한 certificate로 local list를 새로고침할 수 있다.

각각의 client에서 다음의 작업을 실행한다.

```bash
sudo cp ca.crt /usr/local/share/ca-certificates/kubernetes.crt
sudo update-ca-certificates
```

```
Updating certificates in /etc/ssl/certs...
1 added, 0 removed; done.
Running hooks in /etc/ca-certificates/update.d....
done.
```

## Certificate API

`certificates.k8s.io` API를 사용하여 x509 certificates 규정을 인증에 사용한다. [여기](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster)에 문서화되어있다.

