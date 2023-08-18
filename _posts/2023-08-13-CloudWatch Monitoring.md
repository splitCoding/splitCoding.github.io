---
title: "[S-HOOK] AWS CloudWatch를 통한 모니터링 환경을 구축하기"
excerpt: "AWS CloudWatch를 통한 모니터링 환경을 구축하기"
date: 2023-08-18
categories: [s-hook, tech, aws]
tags: [s-hook, cloudwatch, cloudwatch log, cloudwatch agent ]
toc: true
toc_sticky: true
---

# 해결하려는 상황

운영서버에 서비스를 올리기 전에 앞으로 운영하면서 필요한 부분에 대한 실시간 모니터링을 위해 모니터링 환경을 구축해야 한다.

## 모니터링 하려는 대상

- 운영 환경, 개발 환경, 운영 DB 환경
	- 메모리 사용량
	- cpu 사용량 
	- 디스크 사용량
	- 운영환경 실시간 로그

----

## AWS CloudWatch 를 통한 모니터링 구축

CloudWatch 를 사용하면 기본적으로 `AWS/EC2` 네임스페이스 에서 cpu 사용량은 모니터링 할 수 있다.
디스크와 메모리 모니터링은 CloudWatch Agent 를 통해서 모니터링을 할 수 있다.

### CloudWatch Agent 설치

1.  CloudWatchAgentServerPolicy 정책을 가진 IAM 역할을 생성한다.
2. 생성한 IAM 역할을 EC2에 설정한다.
	- 우테코에서 생성해놓은 ec2-cloudwatch-api 라는 IAM에 해당 역할이 포함되어 있을 것 으로 예상된다.
3. EC2에 설치하려는 EC2 환경에 맞는 amazon-cloudwatch-agent.deb 를 다운로드한다.
	- [다운로드 관련 문서](https://docs.aws.amazon.com/ko_kr/AmazonCloudWatch/latest/monitoring/download-cloudwatch-agent-commandline.html)
	- ubuntu, arm64 환경일 때
	
```shell
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/arm64/latest/amazon-cloudwatch-agent.deb
```

4. 다운 받은 deb 파일을 통해서 패키지를 설치한다.

```shell
sudo dpkg -i -E ~/amazon-cloudwatch-agent.deb
```

- 잘못된 deb 파일로 패키지를 생성했다면 아래 명령어로 모든 패키파일과 함께 삭제할 수 있다.

```shell
sudo apt-get remove --purge amazon-cloudwatch-agent
```

5. cloudwatch agent 구성 파일을 생성한다.

선택 1. amazon-cloudwatch-agent-config-wizard 를 통해서 생성한다. [공식 문서](https://docs.aws.amazon.com/ko_kr/AmazonCloudWatch/latest/monitoring/create-cloudwatch-agent-configuration-file-wizard.html)

```shell
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

선택 2.`/opt/aws/amazon-cloudwatch-agent/bin/` 에 직접 config.json 을 생성한다. [공식 문서](https://docs.aws.amazon.com/ko_kr/AmazonCloudWatch/latest/monitoring/CloudWatch-Agent-Configuration-File-Details.html)

6. cloudwatch agent 를 구성파일과 함께 시작한다. [공식 문서](https://docs.aws.amazon.com/ko_kr/AmazonCloudWatch/latest/monitoring/install-CloudWatch-Agent-commandline-fleet.html#start-CloudWatch-Agent-EC2-commands-fleet)

- Linux 환경일 경우 아래 명령어로 시작한다.
```shell
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -s -c file:configuration-file-path
```

- cloudwatch agent 를 재시작해야 하는 경우에는 아래 명령어를 사용한다.

```shell
sudo systemctl restart amazon-cloudwatch-agent
```

#### 사용한 config.json 설정

##### 운영 서버

```json
{
	"agent": {
		"metrics_collection_interval": 10,
		"run_as_user": "root"
	},
	"logs": {
		"logs_collected": {
			"files": {
				"collect_list": [
					{
						"file_path": "/etc/log/app/**.log",
						"log_group_name": "shook-applicaton-log",
						"log_stream_name": "{instance_id}",
						"retention_in_days": 30
					},
					{
						"file_path": "/etc/log/warn/**.log",
						"log_group_name": "shook-warn-log",
						"log_stream_name": "{instance_id}",
						"retention_in_days": 30
					},
					{
						"file_path": "/etc/log/error/**.log",
						"log_group_name": "shook-error-log",
						"log_stream_name": "{instance_id}",
						"retention_in_days": 30
					}
				]
			}
		}
	},
	"metrics": {
		"namespace": "2023-shook-prod",
		"metrics_collected": {
			"disk":{
				"measurement":[
					"disk_total",
					"disk_used",
					"disk_used_percent"
				],
				"resources":[
					"/"
				]
			},
			"mem": {
				"measurement": [
					"mem_total",
					"mem_used_percent"
				],
				"metrics_collection_interval": 10
			},
			"swap": {
				"measurement": [
					"swap_used_percent"
				],
				"metrics_collection_interval": 10
			}
		}
	}
}
```

##### 운영 DB

```json
{
	"agent": {
		"metrics_collection_interval": 10,
		"run_as_user": "root"
	},
	"metrics": {
		"namespace": "2023-shook-prod-db",
		"metrics_collected": {
			"disk":{
				"measurement":[
					"disk_total",
					"disk_used",
					"disk_used_percent"
				],
				"resources":[
					"/"
				]
			},
			"mem": {
				"measurement": [
					"mem_total",
					"mem_used_percent"
				],
				"metrics_collection_interval": 10
			},
			"swap": {
				"measurement": [
					"swap_used_percent"
				],
				"metrics_collection_interval": 10
			}
		}
	}
}
```

##### 개발 서버

```json
{
	"agent": {
		"metrics_collection_interval": 10,
		"run_as_user": "root"
	},
	"metrics": {
		"namespace": "2023-shook-dev",
		"metrics_collected": {
			"disk":{
				"measurement":[
					"disk_total",
					"disk_used",
					"disk_used_percent"
				],
				"resources":[
					"/"
				]
			},
			"mem": {
				"measurement": [
					"mem_total",
					"mem_used_percent"
				],
				"metrics_collection_interval": 10
			},
			"swap": {
				"measurement": [
					"swap_used_percent"
				],
				"metrics_collection_interval": 10
			}
		}
	}
}
```


### 대시보드 구성 [(공식 문서)](https://docs.aws.amazon.com/ko_kr/AmazonCloudWatch/latest/monitoring/create_dashboard.html)


1. 대시보드를 생성한다.
2. 원하는 위젯을 추가한다. [(공식 문서)](https://docs.aws.amazon.com/ko_kr/AmazonCloudWatch/latest/monitoring/create-and-work-with-widgets.html)

#### CloudWatch Agent 가 로그를 모니터링하는 경우

- 아래 로그 그룹에서 CloudWatch Agent 설정에서 생성한 로그 그룹이 있는지 조회한다.

![]({{"/assets/img/cloudwatch/20230818090000.png" | relative_url }})

- 위젯에서 로그테이블을 선택한 뒤 설정한 로그 그룹을 선택해서 추가한다.

![]({{"/assets/img/cloudwatch/20230818093642.png" | relative_url }})




