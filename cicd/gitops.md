# GitOps란?

* git 을 중심으로 CI, CD를 달성하는 운영기법
* k8s 기반이라면 yml 파일들을 기반으로 CI, CD를 자동화
* gitlab을 쓴다면 자체적으로 지원하는 CI/CD 를 쓸 수 있고
* repo는 원하는걸 쓰고 jenkins로 CI 하고 (빌드, 테스트, 컨테이너 빌드 및 push 까지) argoCD와 같은 GitOps 전문 도구로 CD 하는 방법도 있음
* 장점은 인프라스터럭쳐 레벨까지 코드로 관리가 가능해지고 (IaC, Infrastructure as a Code) git을 사용하게 되면서 PR을 통해 코드리뷰도 자연스럽게 수행가능