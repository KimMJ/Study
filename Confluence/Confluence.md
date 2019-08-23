# Confluence TroubleShooting

* 만들기 - 문제 해결 문서
* 제목 : 어떤 현상이 발생하였는지
* 태그 : 이에 대한 태그들
* 내용 : 문제 / 해결방법으로 구분지어서 활용

-------

[troubleshooting을 프로처럼 작성하기](https://confluence.atlassian.com/confcloud/blog/2016/07/write-troubleshooting-articles-like-a-pro)

### Choose a title and label

* 대부분의 사람들은 문제 해결 문서를 검색을 통해 찾는다.
* 따라서 어떤 문제를 해결하는지를 제목으로 정하는 것이 좋다.
* 문서에 라벨을 추가해서 검색과 구조를 쉽게 만들어라.
  * 사용자는 라벨을 검색해서 찾을수도 있고 [Content by Label macro](https://confluence.atlassian.com/display/ConfCloud/Content+by+Label+Macro)를 이용해서 특정 라벨과 관련된 모든 문서의 리스트를 뽑을수도 있다.

### Describe the problem and symptoms

* 사용자가 경험할만한 문제점을 가능한 모호하지 않고 명확하게 설명하라.

### Possible causes and solutions

같은 현상에 대해서 여러 원인이 있고 각각에 대해 솔루션이 다르다면 테이블이나 각각 다른 섹션을 가지도록 만들어라.

* 이 경우 [Table of Contents macro](https://confluence.atlassian.com/confcloud/table-of-contents-macro-724765294.html)를 사용해서 독자로 하여금 찾기 쉽도록 하라.

##### Guide

* 이 섹션에서 작성할 때, 간단하게 해라. 부제목과 항목을 가능하다면 사용하여 따라할 수 있도록 해라.
* 모든 과정을 컴포넌트로 나누어라. 예를 들어 "공간의 관리자가 누군지 확인해라" 보다는 "컨플루언스 헤더에서 **공간** > **공간 목록**를 선택하고 공간 옆에 **공간 상세정보** 아이콘을 를 선택해라. 공간 관리자가 그곳에 적혀있을 것이다."라고 적어라.
* 용어 사용을 피해라. 만일 피할 수 없다면 [Panel macro](https://confluence.atlassian.com/display/ConfCloud/Panel+Macro)나 [Info macro](https://confluence.atlassian.com/display/ConfCloud/Info%2C+Tip%2C+Note%2C+and+Warning+Macros)를 사용하여 정의하거나 용어들에 대한 하이퍼링크를 사용하여 그것들에 대한 자세한 정보가 있는 페이지를 연결해라.
* 스크린샷, 다이어그램, 그림들을 이용하여 문서를 읽기 쉽도록 하고 몇몇 스텝들을 명확하게 해라.

### Related content

문서 말미에 관련 컨텐츠를 추가해서 더 유용하도록 만들어라.

* 비슷하거나 관련된 이슈들

  * [Content by Label macro](https://confluence.atlassian.com/confcloud/content-by-label-macro-724765178.html)를 이용해서 비슷하거나 관련된 이슈를 찾도록 해라.

* 팁과 트릭

  * 다시 해당 현상이 발생하지 않도록 하는 방법을 알고 있다면 팁과 트릭들을 [Tip macro](https://confluence.atlassian.com/confcloud/info-tip-note-and-warning-macros-724765216.html)로 공유해라.

* JIRA 앱을 사용한다면

  * [JIRA Issues macro](https://confluence.atlassian.com/doc/jira-issues-macro-139380.html)를 문제 해결 문서에 추가하여 알려진 이슈에 빠르게 접속하도록 해라.
  * 이는 이슈가 해결되거나 상태가 변화되면 자동으로 업데이트 되도록 한다.

  ![Screen+Shot+2016-07-12+at+4.46.54+PM](img\Screen+Shot+2016-07-12+at+4.46.54+PM.png)

* [**Questions for Confluence**](https://confluence.atlassian.com/questions/questions-for-confluence-documentation-407724474.html)를 사용한다면

  * [Questions list macro](https://confluence.atlassian.com/questionscloud/integrate-questions-with-your-spaces-738361805.html)를 문제 해결 문서에 추가하여 같은 토픽에 대한 질문들을 정렬하여 하이라이트하고 질문하기 버튼을 기본 홈페이지에 추가하라.

* 배포 후

  * 공유 후에는 사용자가 inline 댓글을 사용하여 좀 더 명확한 정보를 줄수도 있다.

### Appendix

* blueprint에 대해서는 알아볼 필요가 있을듯.



