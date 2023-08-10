---
title: "[S-HOOK] 팀 블로그에 작성한 로그 설정 글"
excerpt: "[S-HOOK] 팀 블로그에 작성한 로그 설정 글"
date: 2023-08-10
categories: [s-hook, logging]
tags: [logging, slf4j]
toc: true
toc_sticky: true
---

[원본 글로 이동](https://velog.io/@shook2023/SHOOK-Logging-%ED%99%98%EA%B2%BD-%EC%84%A4%EC%A0%95)


> 안녕하세요 SHOOK 팀의 스플릿입니다 😄


S-HOOK이 실제 사용자들에게 S-HOOK 서비스를 제공하기 위한 운영 환경 배포를 앞두고 있습니다 🙇

제일 중요한 운영환경인 만큼 **항상** 모니터링하고 **빨리** 대처하고자 우선 개발환경에 Logging 환경을 구축해보았습니다 🫡

---

저희 S-HOOK 팀 개발자들이 앞으로 서비스를 운영하면서 로그를 통해 하려는 것들입니다.


- 🖥️ 서비스 실시간 모니터링
- 💊 빠른 문제 해결을 위한 파악
- 📝 애플리케이션의 성능 향상을 위한 정보 수집
- 🫂 사용자의 행동과 패턴을 파악하여 사용자 경험 향상


---


# Logging 구축

## SLF4J ( Simple Logging Facade For Java )

자바를 위한 간단한 로깅 퍼사드

>### SLF4J가 퍼사드인 이유?
>
>SLF4J는 실제 로깅 프레임워크 없이는 돌아가지 않는 인터페이스 역할의 라이브러리이기 때문입니다.

>### SLF4J를 사용하는 이유
>
- 저희가 사용하는 Spring Boot 3.1.1 에서는 spring-boot-starter 에 SLF4J 가 기본으로 있을 정도로 많이 쓰이는 라이브러리 입니다.
![](https://velog.velcdn.com/images/shook2023/post/5eef47f9-c43f-431b-8f64-5675436b5eee/image.png)
- 인터페이스 제공으로 인해 실제 작동하는 로깅 프레임워크가 변경이 되어도 코드에는 변경이 없습니다.

---

## Logback

Logback은 log4j를 기반으로 SLF4J 인터페이스의 구현하여 만든 로깅 프레임워크입니다.

>### Logback을 사용하는 이유
>
- SLF4J 처럼 spring-boot-starter 에 기본으로 있는 많이 쓰이는 라이브러리 입니다.
- SLF4J 와 같이 많이 쓰이는 간단한 로깅 프레임워크로 레퍼런스가 많습니다.
- 더 많은 기능을 가지고 있는 로깅 프레임워크가 아직 필요가 없습니다.
>

---

## 로깅 환경 설정

- 로깅 환경은 운영 서버에서만 작동하려고 합니다.

- 로깅을 하려는 방식은 4가지 입니다.
  - 콘솔 출력
  - 전체 로그를 지속적으로 파일 저장
  - warn 로그를 지속적으로 파일 저장
  - error 로그를 지속적으로  파일 저장
  
- 로그파일을 매일매일 저장하려고 합니다.

- 로그파일들을 위한 용량은 현재 EC2의 남은 용량 중 3GB를 할당하려고 합니다. 

- 로그를 아래 형식의 문자열로 남기려고 합니다. 
 `[시간] 레벨 -- [쓰레드] 로그발생객체[메서드:라인] 메시지 `

---

## 설정대로 작성한 logback.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

  <!-- 스프링 profile 설정 -->
  <springProfile name="prod">

    <!-- 콘솔 출력 appender -->
    <appender name="console" class="ch.qos.logback.core.ConsoleAppender">
      <encoder>
        <pattern>
          [ %d{yyyy/MM/dd HH:mm:ss.SSS} ] %-5level-- [%thread][%logger][%class][%method:%line] - %msg %n
        </pattern>
      </encoder>
    </appender>

    <!-- 애플리케이션 로그 파일 appender -->
    <appender name="file-application" class="ch.qos.logback.core.rolling.RollingFileAppender">
      <file>/etc/log/app/application.log</file>
      <encoder>
        <pattern>
          [ %d{yyyy/MM/dd HH:mm:ss.SSS} ] %-5level-- [%thread] %logger[%method:%line] - %msg %n
        </pattern>
        <charset>utf8</charset>
      </encoder>
      <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
        <fileNamePattern>application.%d{yyyy-MM-dd}-%i.log</fileNamePattern>
        <maxFileSize>100MB</maxFileSize>
        <maxHistory>15</maxHistory>
        <totalSizeCap>2GB</totalSizeCap>
      </rollingPolicy>
    </appender>

    <!-- WARN 로그 파일 appender -->
    <appender name="file-warn" class="ch.qos.logback.core.rolling.RollingFileAppender">
      <file>/etc/log/warn/warn.log</file>
      <encoder>
        <pattern>
          [ %d{yyyy/MM/dd HH:mm:ss.SSS} ] %-5level-- [%thread] %logger[%method:%line] - %msg %n
        </pattern>
        <charset>utf8</charset>
      </encoder>
      <filter class="ch.qos.logback.classic.filter.LevelFilter">
        <level>WARN</level>
        <onMatch>ACCEPT</onMatch>
        <onMismatch>DENY</onMismatch>
      </filter>
      <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
        <fileNamePattern>warn.%d{yyyy-MM-dd}-%i.log</fileNamePattern>
        <maxFileSize>100MB</maxFileSize>
        <maxHistory>15</maxHistory>
        <totalSizeCap>1GB</totalSizeCap>
      </rollingPolicy>
    </appender>

    <!-- ERROR 로그 파일 appender -->
    <appender name="file-error" class="ch.qos.logback.core.rolling.RollingFileAppender">
      <file>/etc/log/error/error.log</file>
      <encoder>
        <pattern>
          [ %d{yyyy/MM/dd HH:mm:ss.SSS} ] %-5level-- [%thread] %logger[%method:%line] - %msg %n
        </pattern>
        <charset>utf8</charset>
      </encoder>
      <filter class="ch.qos.logback.classic.filter.LevelFilter">
        <level>ERROR</level>
        <onMatch>ACCEPT</onMatch>
        <onMismatch>DENY</onMismatch>
      </filter>
      <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
        <fileNamePattern>error.%d{yyyy-MM-dd}-%i.log</fileNamePattern>
        <maxFileSize>100MB</maxFileSize>
        <maxHistory>15</maxHistory>
        <totalSizeCap>1GB</totalSizeCap>
      </rollingPolicy>
    </appender>

    <!-- 로그 root 레벨 & appender 설정 -->
    <root level="info">
      <appender-ref ref="console"/>
      <appender-ref ref="file-application"/>
      <appender-ref ref="file-warn"/>
      <appender-ref ref="file-error"/>
    </root>

  </springProfile>

</configuration>
```
---

## 사용한 logback.xml 설정들에 대한 설명

`<springProfile name="{프로파일명}">`
- 스프링 프로파일에 따른 설정

`<appender name="{어팬더-이름}" class="{어팬더-타입}">`

- appender 를 생성한다.
- 설정한 어팬더 타입을 통해 설정한 이름의 appender 를 생성한다.

`<encoder>`
- appender 에서 사용할 encoder 를 설정한다.

`<pattern>`
- encoder 에서 사용할 pattern 을 설정한다.

>사용한 패턴 형식 목록
- `%d`: 로그 이벤트 날짜, 시간 출력
- `%level`: 로그 레벨 출력 (ex. TRACE, DEBUG, INFO .. )
- `%-5level` : 최대 길이인 5로 출력 영역을 고정하여 로그의 출력 형식 동일하게 지정
- `%thread`: 로그 이벤트 생성 스레드 이름 출력
- `%logger`: 로그 이벤트를 관리한 객체 이름 출력
- `%method`: 로그 이벤트 생성 메서드 이름 출력
- `%line`: 로그 이벤트 생성 라인 번호 출력
- `%msg`: 실제 로그 메시지
- `%n` : 줄바꿈 문자 출력

`<file>`
- 파일로 저장하는 appender 설정에서 사용가능 하며 저장할 파일 위치를 설정한다.
- 절대경로를 사용할 경우 미리 디렉토리를 생성해야놔야 한다.
- 로그파일 저장을 위해 어플리케이션을 실행한 사용자에게 디렉토리에 대한 권한이 있어야 한다.

`<filter class="{사용할 filter 타입}">`
- appender 에서 필터링을 위한 filter 를 설정한다.

`<level>`
- LevelFilter 에서 필터링할 레벨을 설정한다.

`<onMatch>`
- level에 해당 될 때 동작 방식을 설정한다.

`<onMismatch>`
- level 에 해당되지 않을 때 동작 방식을 설정한다.

`<rollingPolicy class="{사용할 rollingPolicy 타입}">`
- RollingFileAppender 에서 사용할 rollingPolicy 를 설정한다.

`<fileNamePattern>`
- RollingFileAppender 에서 저장될 로그파일의 이름을 설정한다.

>사용한 fileNamePattern 형식
- `%d`: dateTime 을 뜻한다.
	- default : { yyyy-MM-dd } :  매 일 자정에 새로운 로그로 rollover
	- %d{yyyy-MM-dd_HH-mm} : 매 분 새로운 로그로 rollover
	- %d{yyyy/MM} : 매 월 새로운 로그로 rollover
- `%i`: SizeAndTimeBasedRollingPolicy 를 사용할 때 로그가 최대크기가 될 경우 자동으로  인덱스를 사용하여 로그 파일을 저장해주는 옵션 ( 없을 시 에러가 난다. )

`<maxFileSize>`
- 한 로그 파일 크기의 최대치를 설정한다.

`<maxHistory>`
- 로그 기록 파일의 총 개수의 최대치를 설정한다.

`<totalSizeCap>`
- 로그 파일들의 총 크기의 최대치를 설정한다.

---

## 설정 전체 설명

- 해당 설정은 스프링의 profile 이 prod 일 때만 적용됩니다.
- 모든 로그는 INFO 레벨 이상 부터 출력됩니다.

- 로그는 아래 형식으로 출력됩니다.
```
[ %d{yyyy/MM/dd HH:mm:ss.SSS} ] %-5level-- [%thread] %logger[%method:%line] - %msg %n
```

- 콘솔에 출력하는 ConsoleAppender 타입의 appender 를 추가했습니다.

- 로그 파일을 저장을 위해 RollingFileAppender 타입의 appender 들을 추가했습니다.

> 로그 파일 저장 방식은 다음과 같습니다.
- utf-8 형식으로 저장합니다.
- 로그 파일은 매일 자정 저장됩니다.
- 한개의 로그파일의 크기가 100MB을 넘는 경우 저장됩니다.

> 로그 파일 삭제 방식은 다음과 같습니다.
- 분류별로 존재하는 로그파일들의 총 크기가 1GB일 때 오랜된 로그를 삭제합니다.
- 분류별로 존재하는 로그파일들의 갯수가 15개를 넘을 때 오래된 로그를 삭제합니다.

> 로그 파일 저장 위치는 다음과 같습니다.
- 모든 로그 저장 위치 : `/etc/log/app/application.log`
- WARN 로그 저장 위치 : `/etc/log/warn/warn.log`
- ERROR 로그 저장 위치 : `/etc/log/error/error.log`

> 로그 파일 저장시 파일명은 다음과 같습니다.
- 모든 로그 파일명  : `application.%d{yyyy-MM-dd}-%i.log`
- WARN 로그 파일명 : `warn.%d{yyyy-MM-dd}-%i.log`
- ERROR 로그 파일명 : `error.%d{yyyy-MM-dd}-%i.log`
>

---

## 마무리

S-HOOK 은 위의 방식으로 로깅 설정을 적용할 계획입니다!!😄

앞으로 S-HOOK 서비스를 운영해 나가면서 서버의 상태, 운영에 필요하다고 여겨지는 로그의 양 등등 여러 변화가 있을 것이라 예상합니다.

그럴 때마다 로깅 설정 또한 상황에 맞춰 바꿔나갈 계획이니 바꾸게 되는 그 날 다시 로깅 관련 글로 돌아오겠습니다. 🙇

긴 글 읽어주셔서 감사합니다 ❤️

