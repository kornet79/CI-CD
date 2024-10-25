---[**노션**](https://incredible-guan-388.notion.site/AWS-Code-Pipeline-CI-CD-12726f5391b18005a229f39bf16bf25d)---
<br>

![aws 이미지](https://imgur.com/k7V7iV6.png)


# **AWS Code Pipeline CI/CD**

# 구성도



![CI/CD 이미지](https://devio2023-media.developers.io/wp-content/uploads/2022/11/CI_CD-1-1536x907.png)






**Github 에 소스 코드가 푸시되면 자동으로 배포되도록 구성되어 있으며 다음과 같은 워크플로우로 진행됩니다.**

1. **유저가 깃 허브에 코드를 푸시하면 Code Pipeline에서 이를 트리거로 인식하여 작동합니다**
2. **소스 코드를 Code Build 에서 빌드하고 아티팩트를 S3에 저장합니다**
3. **아티팩트 저장까지 문제없이 끝났다면 Deploy에서 아티팩트를 확인하고 배포를 준비합니다**
4. **준비가 완료되면 EC2에 배포합니다**

---

### 조건

- **이번 글에서 사용하는 소스 코드는 다음 공식 문서의 코드를 그대로 사용합니다. 따라서 글을 그대로 따라한다면 Maven 환경을 구축할 필요가 있습니다.**
    - [**CodeDeploy sample for CodeBuild**](https://docs.aws.amazon.com/codebuild/latest/userguide/sample-codedeploy.html)
    - **소스 코드까지 똑같이 따라할 필요는 없으므로 변화를 확인할 수 있는 정도의 소스 코드만 준비하면 됩니다.**
    - **단 빌드 후에 appspec.yml이 아티팩트에 포함되어 있어야 합니다**
- **배포할 대상이 EC2 인 경우에는 필요한 인스턴스 프로필을 설정해줍니다**
    - **Code Deploy 관련 IAM [공식 문서](https://docs.aws.amazon.com/ko_kr/codedeploy/latest/userguide/getting-started-create-iam-instance-profile.html)**
    - **Code Deploy Agent를 주기적으로 업데이트 하거나 이번 작업을 하며 System Managers로 인스톨 하는 경우에는 EC2가 System Managers로 컨트롤 가능한 상태여야 합니다. 자세한 내용은 [공식 문서](https://docs.aws.amazon.com/codedeploy/latest/userguide/codedeploy-agent-operations-install-ssm.html)를 참고해주세요.**
- **배포 대상을 구분할 수 있는 태그가 부여되어 있어야 합니다**
- **Github를 이용할 수 있는 환경이 구성되어 있어야 합니다**
    - **Github의 리포지토리에 이미 소스 코드가 푸시되어 있다고 가정하고 작업을 진행합니다.**
- **아티팩트 등을 저장할 S3 버킷이 생성되어 있어야 합니다**
    - **권한 등은 따로 설정할 필요가 없습니다**
- **콘솔 작업을 전제로 합니다**

---

### **만들어보기**

**각 Code 서비스의 콘솔에서 Code Build나 Code Deploy를 작성해도 되지만 이번에는 Code Pipeline에서 작성해보도록 하겠습니다.**

---

### **Code Build 프로젝트 생성**

![대체 텍스트](https://i.imgur.com/N1AGN2o.png)



![대체 텍스트](https://i.imgur.com/omhupCc.png)



**깃 허브를 소스 프로바이더로 지정하였지만 필요하다면 S3나 Code Commit 등 필요한 서비스를 지정해줍니다. 이후에 파이프라인에 소스 스테이지를 추가하기 때문에 웹 훅은 체크를 해제합니다.**

**환경(Envirionment)는 필요한 빌드 서버의 OS를 선택해줍니다.**

**아래의 추가 설정(Additional configuration)에서 서버의 스펙이나 타임 아웃 등의 옵션도 선택할 수 있습니다. 스펙에 따라 [요금](https://aws.amazon.com/ko/codebuild/pricing/)이 다르니 확인하고 적절한 스펙을 선택하도록 합니다.**

**저는 다음과 같이 설정하였습니다.**

![대체 텍스트](https://i.imgur.com/4BPvQyW.png)


### 아티팩트

![대체 텍스트](https://i.imgur.com/So2jKel.png)


**난 따로 S3를 만들고 지정하지 않았기 때문에 없다는걸 확인할수 있다.**

(추후 추가함)

---

### **Code Deploy 어플리케이션 생성**

![대체 텍스트](https://i.imgur.com/dY1Nnsj.png)


---

**Code Deploy의 어플리케이션을 생성합니다.**

**우선 배포에 사용할 IAM 역할을 생성합니다.
IAM 콘솔에서 역할에 들어간 후 역할을 생성합니다.
케이스에서 CodeDeploy를 선택합니다.**

![대체 텍스트](https://i.imgur.com/9xalz95.png)


---

**필요한 권한은 설정되어 있습니다.
마지막에 역할 이름만 설정해줍시다.**

**그리고 Code Deploy 콘솔의 어플리케이션에서 어플리케이션을 생성합니다.
어플리케이션 이름과 배포 할 대상만 선택하면 됩니다.**

**저는  Developer 라는 역할을 생성했습니다.**

**정책은 아래와 같습니다.**

- code_정책
    
    ```jsx
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "",
                "Effect": "Allow",
                "Principal": {
                    "Service": [
                        "codedeploy.amazonaws.com"
                    ]
                },
                "Action": [
                    "sts:AssumeRole"
                ]
            }
        ]
    }
    ```
    

![대체 텍스트](https://i.imgur.com/Uq4KofO.png)


---

**간단하게 APP 만들어주고 배포 그룹 만들어줄개요.**

![대체 텍스트](https://i.imgur.com/Uq4KofO.png)

**이후 배포 그룹을 생성합니다.**

**배포 그룹 이름과 방금 생성한 역할을 설정합니다.**

**그리고 배포할 환경과 방식, 배포 대상이 EC2인 경우 Agent의 인스톨, 업데이트 스케줄 등을 설정합니다.**

**저는 다음과 같은 설정이 되었습니다.**

---

### 역할과 이름

![대체 텍스트](https://i.imgur.com/zKtqjVp.png)

### 배포 유형

![대체 텍스트](https://i.imgur.com/zrw6Osu.png)

**나 같은경우 아직 ec2 설정을 다하지 못한 상태였다.**

**키는 원래 있던 Name , 서버는 ec2중 리소스 남은거 하나 만들어줬다.**

### **AWS Systems Manager를 사용한 에이전트 구성**

![image.png](https://imgur.com/oNN7kqK.png)

### **배포 설정, 로드 밸런스 설정**

---

# 만약 LB 설정과 타겟 그룹을 설정하지 못했다면?

### 만약 LB, 타겟 그룹을 설정하지 못했다면 이 글을 따라해보자

## 타겟 그룹 설정

![image.png](https://imgur.com/zMgMNmu.png)

---

![image.png](https://imgur.com/xu0sFU8.png)

**어떤 LB 사용할껀지 부터 정하고 가자 작자 본인은 ALB 인스턴스 Auto Scaling을 사용하여 EC2의 크기와 숫자를 조절한다는 가정하에 진행하겠다.**

![image.png](https://imgur.com/r77JZ19.png)

**타겟 그룹의 이름을 정의해주고 네트워크 설정을 해줬다.**

**나는 기본 VPC 를 사용하였고 프로토콜 버전은 HTTP1 버전을 사용하였다.**

---

![image.png](https://imgur.com/2wl5F7j.png)

**이제 할당할 인스턴스를 골라주고 그 인스턴스에서 열어줄 port를 입력해준다. 작자는 http 80을 열어주었다.**

![image.png](https://imgur.com/152JQKT.png)

**고롬 요로코롬 타겟 그룹이 생성이 완료 되었다.**

## LB 설정

![image.png](https://imgur.com/MM43rzj.png)

### 이름과 IP 주소 유형 정해줍니다.

---

![image.png](https://imgur.com/PBBNuXw.png)

---

### 다음 네트워크, 보안 그룹 부분을 설정

![image.png](https://imgur.com/KeXibIP.png)

### **리스너 및 라우팅**

![대체 텍스트](https://i.imgur.com/W39J7tM.png)

아까 타겟 그룹을 만들고 지정해줘야 한다.

태그는 선택사항 나는 개인적으로 추천 (분류 편함)

![대체 텍스트](https://i.imgur.com/FH89llY.png)


요약 검토 하고 생성

![대체 텍스트](https://i.imgur.com/ukJSZZJ.png)

그럼 완성!

---

![대체 텍스트](https://i.imgur.com/mYnFMUp.png)


**나는 활성화 해주고 아까 만든 LB설정 해주고 만들어 줬다.**

---

### 어플리케이션 완성.

![image.png](https://imgur.com/F3qSfLG)

# Code Pipeline 생성

![대체 텍스트](https://i.imgur.com/Vqj3W3Y.png)

---

Code Pipeline 콘솔에서 파이프라인 작성을 클릭합니다.

우리는 우리가 하나하나 만들거라 사용자 정의 해주자.

---

![대체 텍스트](https://i.imgur.com/MDoHvw3.png)


---

### 우선 파이프라인의 이름과 역할을 작성합니다.

---

![대체 텍스트](https://i.imgur.com/4YQqQtk.png)


---

### 소스 연결

---

![image.png](https://imgur.com/ZRGrusL.png)

### **빌드 스테이지 추가**

---

![image.png](https://imgur.com/V2HivRI.png)

**나는 그냥 hello world 출력문 달아줬다.**

### 배포 스테이지

---

![image.png](https://imgur.com/X5jOgRH.png)

**아까 우리가 만든  APP, 타겟 그룹이 잘살아있는것을 확인할수 있다.**

- 추가 확인 사항
    
    여기까지 설정이 잘 끝났다면 다음을 눌러 내용을 확인한 후에 문제가 없다면 파이프라인을 생성합니다.
    문제없이 끝났다면 배포까지 한번씩 실행됩니다. 배포를 원치 않는다면 중지를 눌러주세요.
    
    배포 대상에도 제대로 반영이 되었는지 확인해봅니다.
    

## 파이프 라인 완성

![image.png](https://imgur.com/X5jOgRH.png)

## **깃 허브에 코드 업로드해보기**

![image.png](https://imgur.com/u7Q5JYa.png)

**전체적인 파이프라인에 문제가 없다면 마지막으로 깃 허브에 코드를 수정하여 푸시해봅시다.**

**파이프라인이 웹 훅을 읽고 배포까지 문제없이 진행된다면 이번 작업은 완료입니다.**

# 마무리

---

**테스트 코드가 필요한 경우에는 Code Build에 적용할 수 있으니 필요에 따라 이용하도록 합니다.**

**긴 글 읽어주셔서 감사합니다.**

**오탈자 및 내용 피드백은 언제나 환영합니다. [kornet321@naver.com](mailto:kornet321@naver.com) 로 연락 주시면 감사합니다!**

**압도적인 블로그들에 대한 감사…**

---

참초 : https://docs.aws.amazon.com/ko_kr/codedeploy/latest/userguide/getting-started-create-iam-instance-profile.html

도움 : https://aws.amazon.com/ko/codepipeline/,

https://cyberx.tistory.com/312,

https://velog.io/@inssg/CI-CD-AWS-Code-Deploy-Code-Pipeline-활용

https://dev.classmethod.jp/articles/create-aws-code-pipeline-ci-cd-environment-kr/
