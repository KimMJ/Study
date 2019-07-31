# NAT(Network Address Traslation)

* IP 패킷의 TCP/UDP 포트 숫자와 소스 및 목적지의 IP주소 등을 재기록하면서 라우터를 통해 네트워크 트래픽을 주고 받는 기술
* 패킷에 변화가 생기기 때문에 IP나 TCP/UDP의 체크섬도 다시 계산되어 재기록해야 한다.
* 보통 사설 네트워크에 속한 여러개의 호스트가 하나의 공인 IP로 인터넷에 접속하기 위해서 사용

| **Full-cone NAT** (*one-to-one NAT*) | [![Full Cone NAT.svg](https://upload.wikimedia.org/wikipedia/commons/thumb/4/44/Full_Cone_NAT.svg/400px-Full_Cone_NAT.svg.png)](https://commons.wikimedia.org/wiki/File:Full_Cone_NAT.svg) |
| ------------------------------------ | ------------------------------------------------------------ |
| **(Address)-restricted-cone NAT**    | [![Restricted Cone NAT.svg](https://upload.wikimedia.org/wikipedia/commons/thumb/3/3c/Restricted_Cone_NAT.svg/400px-Restricted_Cone_NAT.svg.png)](https://commons.wikimedia.org/wiki/File:Restricted_Cone_NAT.svg) |
| **Port-restricted cone NAT**         | [![Port Restricted Cone NAT.svg](https://upload.wikimedia.org/wikipedia/commons/thumb/c/c2/Port_Restricted_Cone_NAT.svg/400px-Port_Restricted_Cone_NAT.svg.png)](https://commons.wikimedia.org/wiki/File:Port_Restricted_Cone_NAT.svg) |
| **Symmetric NAT**                    | [![Symmetric NAT.svg](https://upload.wikimedia.org/wikipedia/commons/thumb/7/73/Symmetric_NAT.svg/400px-Symmetric_NAT.svg.png)](https://commons.wikimedia.org/wiki/File:Symmetric_NAT.svg) |

---------

[https://ko.wikipedia.org/wiki/%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC_%EC%A3%BC%EC%86%8C_%EB%B3%80%ED%99%98](https://ko.wikipedia.org/wiki/네트워크_주소_변환)

