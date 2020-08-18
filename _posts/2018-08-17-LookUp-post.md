---
title: "Look Up Server"
date: 2018-08-17 15:52:00 +0900
# categories: Notice
---

# Look Up Server 개요
## 1. Look Up Server란
* 클라이언트가 게임에 입장 하기전, 자신의 App 정보를 Look Up Server로 보내면 Look Up Server 해당 정보를 바탕으로 특정 정보를 리턴해주는 Server

## 2. Look Up Server 의 목표
* 게임에 입장하기 전, 특정 정보를 리턴을 하는 시스템
* Client App 실행 시 Client App의 Version 과 Bundle Id, Language Code를 분석한다.
* Look Up Server는 분석한 정보를 바탕으로 정보에 맞는 게임서버로 접속을 하도록 유도 한다.

## 3. Look Up Server 요약
* Look Up Server 에서 아래와 같은 기능을 제공한다.
    - Client 의 정보와 Game Server의 상태에 따라 리턴 하는 정보가 결정된다.
    - Client App 의 Version 과 Bundle Id 를 체크한다.
    - 체크 후 접속 할 서버의 정보를 전달한다.
    - 접속 할 서버가 점검 중 이면 Language Code 를 체크한다.
    - 체크 후 Language Code 에 맞는 점검 메시지를 전달한다.


# REST API
## 1. REST API 사전 지식
### 1.1. REST API
* REST 는 **Representational State Transfer** 라는 용어의 약어 이다.
    - 자원을 이름(자원의 표현)으로 구분하여 해당 자원의 상태(정보)를 주고 받는 모든 것을 의미

### 1.2. REST API 구성 요소
* 자원(Resource)
    - 모든 자원에는 고유한 ID가 존재하게 된다. 이 자원은 Server에 존재한다.
    - 자원을 구별하는 ID는 HTTP URL로 구분하게 된다.
    - Client는 URL을 이용하여 자원을 지정하고 해당 자원에 대한 조작을 Server에 요청한다.
* 행위(Verb)
    - Client는 HTTP Method(POST, GET, DELETE, PUT)를 이용하여 지정한 자원에 대한 조작을 요청한다.
* 표현(Representation Of Resource)
    - Client가 Server의 자원에 대한 조작을 요청하면 Server는 이에 대한 적절한 응답(Representation)을 보낸다.
## 2. Client 에서 Look Up Server 호출
### 2.1. Look Up Server의 URL
* 기본 형태
    - http://[DomainName]/[Resource]/[PlatformBundleID]/[AppVersion]/[LanguageCode]
* Example
    - http://VersionServer/Version/com.example.myapp/1.1.2/KR
    - http://VersionServer/Version/com.example.myapp/1.1.2/US

### 2.2. Response
* 응답 Result 는 JSON 형태로 리턴이 됩니다. 
```json
{
    "ReturnValue":1,
    "Data":{}
}
```
* 메시지의 형태는 ReturnValue  와 Data 의 JSON으로 응답이 됩니다.
* ReturnValue에 따라서 Data를 Handling 하는 방법이 달라집니다.
    - ReturnValue
        + 1, OK => 현재 버전이 서비스 중이고 서버도 정상
        + 2, Check => 현재 버전이 점검 중
        + 3, Update => 현재 버전이 서비스 중이지 않고 업데이트 가 필요함
        + 4, Invalid URL => 잘못된 URL로 요청했을 때
        + 5, Internal Server Error => 서버 에러
    - Data
        + 1, OK
            * 현재 서버의 상태가 정상
```json
{ 
    "ServerInfos":[ 
        { 
            "PacketType":1, 
            "ServerIP":"127.0.0.1", 
            "ServerPort":50000 
        } 
    ] 
}
```
        + 2, Check
            * 현재 서버의 상태는 점검 중
```json
{
    "CheckMsg":{
        "StartUtcTime":"2020-08-10T10:00:00Z",
        "EndUtcTime":"2020-08-10T11:00:00Z",
        "Message":"Test Message"
    }
}
```
        + 3, Update
            * 이 타입을 리턴 받으면 업데이트를 진행
            * Data가 없습니다.
        + 4, Invalid URL
            * 잘못된 요청
            * Data가 존재 하지 않습니다.
        + 5, Internal Server Error
            * 서버 내부에서 에러가 발생
            * Data가 존재 하지 않습니다. 

# Server의 상태 전달
## 1. 각 서버들의 상태를 받거나 전달 하기 위한 준비
### 1.1. Redis => pub/sub
* Look Up Server는 서버의 상태변환을 전달 받기 위해 Redis에 LookUp 이라는 채널로 Subscribe 합니다.
* 서버의 상태 변환을 할 수 있는 역활을 맡은 서버(이하 운영툴 이라함) 는 Look Up Server로 상태를 전달 하기 위해 Redis에 LookUp 이라는 채널로 Publish 합니다.
## 2. 각 서버들의 상태를 주고 받기 위한 Protocol Json
### 2.1. 서버상태의 Json 형태
* 운영툴이 Look Up Server로 서버 상태를 알려주기 위한 Json 의 기본 포맷
    - 1, OK 일 때
```json
{
    "Status":1,
    "BundleID":"com.example.myapp",
    "Version":"1.1.2",
    "ServerInfos":[
        {
            "PacketType":1,
            "ServerIP":"127.0.0.1",
            "ServerPort":50000
        }
    ]
}
```
    - 2, Check 일 때
```json
{
    "Status":2,
    "BundleID":"com.example.myapp",
    "Version":"1.1.2",
    "Messages":{
        "KR":{
            "StartUtcTime":"2020-08-10T10:00:00Z",
            "EndUtcTime":"2020-08-10T11:00:00Z",
            "Message":"테스트 메시지"
        },
        "EN":{
            "StartUtcTime":"2020-08-10T10:00:00Z",
            "EndUtcTime":"2020-08-10T11:00:00Z",
            "Message":"Test Message"
        }
    }
}
```
    - 3, Update 일 때
```json
{
    "Status":3,
    "BundleID":"com.example.myapp",
    "Version":"1.1.2"
}
```
* 서버 상태에 맞게 Json을 작성을 해서 운영툴에서는 Redis로 publish 명령어를 사용해서 Look Up Server가 Subscribe 하고 있는 채널 LookUp으로 메시지를 전달한다.