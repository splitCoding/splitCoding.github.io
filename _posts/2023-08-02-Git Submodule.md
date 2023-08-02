---
title: "Git Submodule 사용과 문제 상황"
excerpt: "Git Submodule 사용 방법과 발생 가능한 문제 상황."
date: 2023-08-02
categories: [git]
tags: [ git, git submodule]
toc: true
toc_sticky: true
---

# Git Submodule

서브 모듈을 사용하면 상위 레포지토리 ( super project )에 하위 레포지토리 (서브모듈 ) 를 두어 관리할 수 있게 해준다.

민감한 보안 정보들을 public repository 에서 노출되지 않게 하기위해 private repository 를 서브모듈로 사용하여 관리하는 경우가 많다.

# 설정하는 방법

## 1. 상위 레포지토리를 클론한다. 

![]({{"/assets/img/20230802214716.png" | relative_url}})

## 2. 하위 레포지토리를 등록한다.


```shell
git submodule add {repository 주소}
```

![]({{"/assets/img/20230802215010.png" | relative_url}})
![]({{"/assets/img/20230802215107.png" | relative_url}})

super project 안에 `.gitmodules` 파일과 서브모듈로 추가한 레포지토리 ( submodule )가 생긴다.

`.gitmodules` 파일안에는 서브 모듈의 경로와 url의 정보가 들어있다.

![]({{"/assets/img/20230802215212.png" | relative_url}})

변경사항을 커밋, 푸시한다.
![]({{"/assets/img/20230802215437.png" | relative_url}})
![]({{"/assets/img/20230802215740.png" | relative_url}})


코드와 커밋 내역에 submodule 에 대한 내용은 보여지지 않는다.

![]({{"/assets/img/20230802220005.png" | relative_url}})
![]({{"/assets/img/20230802215853.png" | relative_url}})



### 서브 모듈을 변경한뒤 상위 레포지토리에서 변화를 감지할까?

1. 서브 모듈 내용 변경

![]({{"/assets/img/20230802220357.png" | relative_url}})

2. 상위 레포지토리에서 git status 를 확인하니 변화를 감지하고 있다.

![]({{"/assets/img/20230802220459.png" | relative_url}})

submodule/test.txt 에 대한 변화가 아닌 submodule 디렉토리 자체의 변화로 감지하고 있다.


**반드시 서브모듈에서 커밋, 푸쉬를 진행해야한다. 만약 상위 레포지토리에서만 진행할 경우 해당 변경 사항 날라갈 수 있다.**

( 상위 레포지토리에서도 커밋하고 푸시하는 것을 권장한다. [서브모듈 업데이트 과정에서 문제](#발생-가능한-문제-상황2)가 생길 수 있다. )


## 3. 상위 레포지토리를 삭제하고 다시 클론해보자

서브 모듈을 조회해보자.

![]({{"/assets/img/20230802231140.png" | relative_url}})

조회는 되지만 현재 submodule 디렉토리는 비어있다.

![]({{"/assets/img/20230802223543.png" | relative_url}})

지금 submodule은 .git 도 존재하지 않는 그냥 디렉토리이다.


### 서브모듈을 초기화 한다.

```shell
git submodule init
```

![]({{"/assets/img/20230802224152.png" | relative_url}})

submodule 경로에 서브 모듈이 등록되었다는 문구가 나오지만 아직까지도 submodule 은 그저 디렉토리이다.


### 초기화한 서브모듈을 업데이트 한다.

```shell
git submodule update
```

( 서브모듈 레포지토리에서 제대로 커밋하고 푸쉬했다면 업데이트시 서브모듈이 제대로 업데이트 된다. ) 



---

# 발생 가능한 문제 상황 1

![]({{"/assets/img/20230802224612.png" | relative_url}})

## 문제의 이유

업데이트시 상위 레포지토리는 현재 자신의 상태에서 커밋 정보를 가져와서 서브모듈에 대한  Checkout을 진행하게 된다.

하지만 상위 레포지토리에만 커밋이 존재하는 경우에 이 문제가 발생한다.

( 상위 레포지토리 커밋 기록 위에 로그에 있는 076d68이 보인다. )
![]({{"/assets/img/20230802225350.png" | relative_url}})

저 커밋에서 나는 서브 모듈에 파일을 추가하고 상위 레포지토리에서만 커밋을 했다.

(텅 비어 있는 서브 모듈 레포지토리 커밋 기록)
![]({{"/assets/img/20230802225649.png" | relative_url}})

즉 checkout 할 커밋 정보가 서브 모듈 원격 저장소에는 없는 상황인 것이다. 

내가 서브모듈에 적용한 변경사항이 서브모듈 레포지토리 존재하지 않아 날라간 것이다.

만약 서브모듈의 내용이 있는 로컬 저장소가 있다면 서브모듈 레포지토리에 커밋, 푸쉬하고 아래 과정을 진행하여 복구할 수 있다. ( 없다면..? 재앙이라고 볼 수 있다..)

### 문제 해결을 위해 서브 모듈 삭제 후 다시 추가

삭제하는 방법

```shell
git submodule deinit -f submodule
rm -rf .git/modules/submodule
rm -rf submodule
git commit -m "remove: 서브모듈 삭제"
git push
```


이제 다시 서브모듈을 추가해보자

---


# 발생 가능한 문제 상황2


현재 내 서브모듈에는 test.txt 가 존재 한다.

![]({{"/assets/img/20230802234746.png" | relative_url}})

상위 레포지토리에서 git submodule update 진행해보자

![]({{"/assets/img/20230802234946.png" | relative_url}})

submodule 안에 test.txt 파일이 존재하지 않는다??

![]({{"/assets/img/20230802235037.png" | relative_url}})

## 문제의 이유


현재 브랜치를 한번 잘보자 submodule 에는 존재하지 않는 c2bdd0c 라는 브랜치이다

위에서 말했듯이 상위 레포지토리는 자신의 현재 상태에서 커밋을 가져와 서브모듈에 대한 Checkout 을 진행한다.

( 상위 레포지토리 코드에 보이는 c2bdd0c )
![]({{"/assets/img/20230802235416.png" | relative_url}})

상위 레포지토리에 c2bdd0c를 클릭해서 들어가 보면

![]({{"/assets/img/20230802235509.png" | relative_url}})

이렇게 test.txt 가 없는 상태의 서브모듈 브랜치인 것이다.

( 상위 레포지토리에서 서브모듈의 변화를 커밋 푸쉬하여 상태를 저장하지 않았기 때문이다. )


## 문제를 해결할 수 있는 방법

1. 서브모듈의 사용하려는 브랜치를 바꿔준다.

![]({{"/assets/img/20230803000315.png" | relative_url}})

하지만 해당 방법은 상위 레포지토리에서 `git submodule update` 를 하는 순간 원래로 돌아간다.

2.  `git submodule update --remote` 를 사용한다.

![]({{"/assets/img/20230803000502.png" | relative_url}})

원격저장소의 내용으로 업데이트가 진행된다.


서브모듈의 변화를 상위 레포지토리에서도 항상 커밋, 푸쉬하여 관리한다면  `git submodule update` 로도 문제 없이 서브모듈을 업데이트 할 수 있을 것이다!

---

## 글을 쓴 계기 & 느낀 점

서브모듈을 2번 정도 사용해 보았는데 시간이 촉박하여 서브모듈에 대해 제대로 파악하지 못하고 검색으로 그저 명령어만 찾아서 진행했었다.

그러다보니 두번째 상황을 이번에 우테코에서 겪게 되었는데 --remote 를 통해 해결은 했지만 정확히 어떤 문제인지가 파악되지 않아 혼자 여러실험을 하면서 문제를 파악했다.

첫번째 문제 상황 같은 경우는 일부러 문제를 발생시켜보며 겪은 상황이였는데 실제로 발생했다면 많은 정보가 날라갈 수 있는 상황이였을 것 같아 미리 겪어본 것이 너무 다행이라는 생각을 했다.

