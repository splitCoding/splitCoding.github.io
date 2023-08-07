---
title: "insufficient permission for adding an object to repository database 오류 트러블 슈팅"
excerpt: "insufficient permission for adding an object to repository database 오류 트러블 슈팅"
date: 2023-08-07
categories: [git]
tags: [git]
toc: true
toc_sticky: true
---

# 문제 상황

갑자기 `git pull` 을 진행하는 상황에서 해당 문제가 발생했다.

```shell
error: insufficient permission for adding an object to repository database .git/objects
fatal: failed to write object
fatal: unpack-objects failed
error: Could not fetch origin
```


---

# 해결 과정


## 1. 첫 로그 파악

```shell
insufficient permission for adding an object to repository database .git/objects`
```

레포지토리 데이터베이스 .git/objects 에 object 를 추가하려는데 권한이 충분하지 않다는 내용이다.

.objects 가 뭔지부터 파악해보기 위해 공부해보니 git 의 file system 에 대한 이해가 필요하다 생각하여 깊게 가게 되었다.

---

## 2.  objects 파악을 위한 Git 의 File System 학습

### Git File System

- Key - Value 형태의 데이터 저장소이다.
- 생성된 개체들의 정보들은 .git/objects 디렉토리에 저장된다.
- 모든 개체들은 접근할 수 있는 유일한 해쉬값 ( Key ) 을 가지고 있다.


#### Blob 란?
- 일반적인 파일들을 뜻하는 개체 
#### Tree 란?
- blob을 가지고 있는 디렉토리, 폴더들을 뜻하는 개체
#### Commit 이란?
- 커밋을 뜻하는 개체
- 커밋 : 전체 개체들의 상태들을 저장하는 단위

#### objects 레포지토리 데이터베이스 살펴보기


git init 을 한 상태에서는 아래와 같이 아무것도 없다.

![]({{"/assets/img/20230803132822.png" | relative_url}})


test.txt 파일을 생성후 add 한 뒤 stage 된 파일을 조회한다.

```shell
git ls-files --stage
```
![]({{"/assets/img/20230803140003.png" | relative_url}})


test.txt 개체의 hash값을 볼 수 있다. (1ac509308a521be3f4f167d1c37c9e1cd87c2b65 )

( objects에 1a/c509308a521be3f4f167d1c37c9e1cd87c2b65 가 생겼다. )
![]({{"/assets/img/20230803140813.png" | relative_url}})


이제 commit을 해보자.

( 아래와 같이 objects 안에 1e 와 60 이 추가로 생성되었다. )
![]({{"/assets/img/20230803141437.png" | relative_url}})

( 1e 에 있는 파일은 commit 타입이고 커밋의 내용이 포함되어 있다. )
![]({{"/assets/img/20230803141620.png" | relative_url}})

( 60에 있는 파일은 tree 타입이고 tree 안에 있는 개체에 대한 내용이 나옵니다. )
![]({{"/assets/img/20230803141755.png" | relative_url}})


tree 에 대해 정확히 보기위해서 hello이라는 디렉토리에 hello1.txt, hello2.txt 를 생성한뒤 commit 해보았다.

( 루트 tree 안에 hello 라는 tree가 생성되었고 생성된 tree에 2개의 blob이 들어있다. )
![]({{"/assets/img/20230804201235.png" | relative_url}})


위에서 봐야할 부분이 한군데가 더 있다.
루트 tree 의 해쉬값이 60f26335.... 에서 5e136548.... 로 달라졌다는 것이다. 

( 위에는 현재 상태의 루트 tree 의 정보 / 아래는 이전 상태의 루트 tree 의 정보 )
![]({{"/assets/img/20230804204400.png" | relative_url}})


위와 같이 git은 같은 개체의 여러상태들을 objects 안에서 관리하고 있다.
이를 통해 개체들을 commit 했을 때의 상태로 변경할 수 있다.

#### Pack 디렉토리

팩 디렉토리 안에는 .pack 파일과 .idx 파일이 존재한다.

![]({{"/assets/img/20230804212519.png" | relative_url}})

##### pack 파일

- 여러 개의 객체를 하나의 파일로 압축한 파일
- 하나로 압축하여 저장하여 저장소의 크기를 줄이고 전송 속도를 높여준다.
- 주로 로컬 저장소와 원격 저장소 간의 데이터 전송에 사용된다.
	- 팩 파일을 전송하는 것이 모든 개체를 전송하는 것보다 효율적이다.
	- 원격 저장소에서는 팩 파일을 받아 개체를 복원하여 사용 가능하다.

##### idx 파일

- 팩 파일에 압축되어 저장된 개체들에 대한 정보와 위치는 담은 파일
- 인덱스 파일을 통해 Git은 팩 파일 내에서 객체를 검색하고 접근할 수 있다.

만약 현재 objects 안에 있는 개체들을 packing 하고 싶다면 아래 명령을 사용하자
```shell
//Git Garbage Collector

git gc
```
![]({{"/assets/img/20230804210704.png" | relative_url}})


---

## 3.  나머지 로그 파악

```shell
fatal: failed to write object //object 를 쓰는데 실패
fatal: unpack-objects failed //objects 를 unpack 하는데 실패
error: Could not fetch origin //origin 을 가져오는데 실패
```

결국 개체를 쓰는데 실패했고 패킹된 개체들을 복원하는데 실패한 것으로 보인다.

---

## 4. 전체 로그 파악

권한이 충분하지 못해서 pack을 unpacking 하여 개체들을 복원하여 write 하는데 실패

이 과정이 git pull의 git fetch 에서 발생했고 그렇다면 원격저장소에 받은 pack 파일을 로컬저장소에서 사용하기 위해서 unpacking 하는 과정에서 권한이 없어 실패했을 것이다.

---

## 5. 동일한 로그를 발생시켜 보자


우선 기본 objects 를 포함한 디렉토리들의 권한을 보면 쓰기 권한은 파일 소유자에게만 존재한다.
```
drwxr-xr-x // d : 디렉토리 , 소유자 권한 [rwx], 그룹 권한 [r-x], 외부 사용자 [r-x]
```
![]({{"/assets/img/20230804221356.png" | relative_url}})

소유자가 아닌 경우 해당 object를 write할 권한이 존재하지 않는 것이다.

원격 저장소에서 파일을 수정한뒤 커밋을 진행하였다. 이 부분은 현재 로컬 저장소에 반영된적이 없다.
![]({{"/assets/img/20230804223819.png" | relative_url}})

objects 디렉토리의 권한을 root 로 바꿔주었다.

![]({{"/assets/img/20230807123905.png" | relative_url}})


사용자가 split 인 상황에서 git fetch 를 진행해보자!

![]({{"/assets/img/20230804223649.png" | relative_url}})

문제 상황과 동일한 에러 로그가 출력되었다!!


그러면 이번엔 git pull 을 진행해보자!

![]({{"/assets/img/20230804224635.png" | relative_url}})

역시 동일한 에러 로그가 출력되었다!!

---

## 6. 문제 해결

```shell
error: insufficient permission for adding an object to repository database .git/objects
```


위에 로그는 결국 objects 안에서 쓰기 작업을 진행할 때 권한이 없어 생기는 문제인 것이다.

### 해결법

1. 개체의 소유자로 현재 사용자를 변경하는 법
2. 현재 사용하는 사용자로 개체를 소유자를 교체하는 법
3. 현재 사용자와 소유자와 그룹이 같은 경우 모든 개체들에게 그룹에게 쓰기 권한을 주는 법

여러 곳에서 해당 환경에 접속하여 git 관련 작업이 수행될 경우 소유자가 다른 여러 개체들이 생길 수 있기에 해당 문제가 발생하는 것으로 판단된다.


---

## 느낀점


사실상 본래의 문제 해결보다 디깅을 굉장히 많이 하면서 작성한 글이였다.

제일 디깅 많이한 부분이 .git 의 기본 구조였는데 이 부분에 대해서 너무 재밌게 공부할 수 있어서 좋았다.