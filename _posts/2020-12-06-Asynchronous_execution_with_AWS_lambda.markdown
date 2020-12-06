---
layout: post
title: "Asynchronous execution with AWS lambda"
date: 2020-12-06 19:00:00 +0900
image: 5.jpg
tags: [aws, lambda]
categories: aws
---
{:refdef: style="text-align: center;"}
![lambda](/images/aws/serverless-framework/lambda.png)
{: refdef}


---
## AWS Lambda 특징

AWS Lambda에는 여러가지 특징이 있습니다.

* 사용한 만큼만 요금이 부과됩니다.~~(100ms 당 요금이 부과된다.)~~
   1. 이번에 Lambda billing 시스템에 업데이트 되었네요. 1ms단위로 부과됩니다. [여기](https://aws.amazon.com/ko/about-aws/whats-new/2020/12/aws-lambda-changes-duration-billing-granularity-from-100ms-to-1ms/)를 참조하시면 됩니다.
* 서버 관리가 불필요합니다..
* 각 트리거에 대한 응답으로 코드를 실행하여 애플리케이션을 자동으로 확장하거나 축소가 가능합니다.
* custom docker image를 지원합니다. [여기](https://aws.amazon.com/ko/blogs/aws/new-for-aws-lambda-container-image-support/)를 참조하시면 됩니다.
  
또한 아래와 같은 일들이 가능합니다.


### 실시간 파일 처리
{:refdef: style="text-align: center;"}
![lambda](/images/aws/lambda/realtime-file-run.png)
{: refdef}

Amazon S3를 사용하여 업로드하는 즉시 데이터를 처리하도록 AWS Lambda를 트리거할 수 있습니다. 예를 들어, Lambda를 사용하여 실시간으로 이미지를 썸네일하고, 동영상을 트랜스코딩하고, 파일을 인덱싱하고, 로그를 처리하고, 콘텐츠를 검증하고, 데이터를 수집 및 필터링할 수 있습니다.

### 실시간 스트림 처리
{:refdef: style="text-align: center;"}
![lambda](/images/aws/lambda/stream-lambda.png)
{: refdef}


AWS Lambda 및 Amazon Kinesis를 사용하여 애플리케이션 활동 추적, 트랜잭션 주문 처리, 클릭 스트림 분석, 데이터 정리, 지표 생성, 로그 필터링, 인덱싱, 소셜 미디어 분석, IoT 디바이스 데이터 텔레메트리 및 측정을 위한 실시간 스트리밍 데이터를 처리할 수 있습니다.

### 서버리스 웹앱

{:refdef: style="text-align: center;"}
![lambda](/images/aws/lambda/web-application-lambda.png)
{: refdef}

AWS Lambda를 다른 AWS 서비스와 결합하면, 확장성, 백업 또는 여러 데이터 센터 중복에 필요한 별도의 관리 작업 없이 개발자가 자동으로 확장 및 축소되고 여러 데이터 센터에 걸쳐 가용성이 높은 구성에서 실행되는 강력한 웹 애플리케이션을 구축할 수 있습니다.

---
## 하지만 timeout...
이번에 저는 데이터 수집 및 처리 과정에서 lambda를 많이 사용하였는데요.

사용을 하다 보니 여러 이슈가 있었지만, 가장 힘들었던 점이 timeout을 고려해야 한다는 점이였습니다.

lambda에서 최대 timeout은 ~~5분입니다.~~ [업데이트](https://aws.amazon.com/ko/about-aws/whats-new/2018/10/aws-lambda-supports-functions-that-can-run-up-to-15-minutes/)를 통해 15분으로 timeout이 증가되었습니다. 한 lambda의 작업이 5분 안에는 이루어 져야 한다는 점이죠. 그래서 기존 코드나 script를 그대로 lambda에 옮겨 놓으면 timeout이 발생할 수 있습니다. (제가 이번에 그랬죠..)

물론 lambda의 성능을 높이면 되긴 합니다. 하지만 성능을 높일수록 가격이 높아지죠.
{:refdef: style="text-align: center;"}
![lambda](/images/aws/lambda/pricing.png)
{: refdef}
위 그림은 AWS Lambda 홈페이지의 요금표입니다. 살펴보시면 128MB가 제일 낮은 성능이고, 최고는 3008MB 입니다.

---

## 해결
어떻게 해결을 할까 여러 고민을 하였습니다. lambda에서 하는 일이 ec2로도 가능은 하지만, 서버 관리 시간, 노력이 많이 들어갈 것 같았습니다. 그리고 작업이 이루어지지 않는 시간동안에도 서버가 구동되어 있어야하며 결국 비효율적인 요금이 발생하게 됩니다.

그래서 고민을 하던 중 lambda의 작업들을 세분화로 나누고, 작업을 분산시키기로 하였습니다. 분산된 작업을 하는 lambda를 worker라고 지칭하겠습니다. worker는 데이터를 수집 및 s3에 저장하는 lambda과 저장된 데이터를 parsing하는 lambda로 구성하였습니다.

하지만 이 분산된 lambda를 어떻게 내 입맛에 맞게 호출할 지 참 막막하더군요. 또한 async하게 분산 작업을 하고 싶었습니다.

{:refdef: style="text-align: center;"}
![lambda](/images/aws/lambda/architecture.png)
{: refdef}

그래서 다음과 같은 구조로 데이터 처리가 되게 하였습니다.

AWS Lambda Manager가 있고, 이 manager가 여러 worker lambda instance를 invoke하여 작업을 분산시켰습니다.

그리고 s3 bucket에 데이터를 쌓은 뒤 objectCreated 이벤트에 의해 lambda가 트리거되도록 구성하였습니다.

원래는 producer/consumer 개념으로 중간에 SQS(아마존 큐 서비스)를 이용해볼까 하였지만, 다음과 같은 쉬운 방법이 있었습니다.
(환경은 python 3.6.2 입니다.)

{% highlight python %}
import boto3
import json

lan = boto3.client('lambda')

payload = {}
payload['hello'] = 'hi'

# worker Lambda 호출
lan.invoke(FunctionName="function_name", InvocationType='Event', Payload=json.dumps(payload))
{% endhighlight %}

boto3 라이브러리의 invoke함수를 이용하는 방법입니다. (boto3는 Python AWS SDK입니다.) function name에는 lambda 함수명을 넣어주면 되구요. InvocationType같은 경우는 Event로 주었습니다. (Sync하게 구성을 하고 싶으시면 RequestResponse 옵션을 주시면 됩니다.)

그리고 event로 넘겨줄 데이터를 dictionary 변수에 넣어주고 json형태로 넘겨주면 됩니다. 자세한 것은 링크에서 확인하시면 됩니다.

저 같은 경우 위 방식을 적절하게 응용하여서 lambda instance를 여러 개 생성하여 분산 처리를 하였습니다.

---

## 마무리
위 방법을 적용하였더니, timeout문제는 전혀 없었습니다. 역시 잘 쪼개야 하더군요.ㅎㅎ

Lambda를 쓰면서 가장 좋았던 점은 서버 관리를 따로 하지 않아도 된다는 점이였고, 또한 매우 싸다는 점입니다. 저 같은 경우는 10분단위로 작업들이 몇가지가 돌아가는데, 계산을 해보니 월 0.8달러 정도면 가능하였습니다. 기존 ec2를 사용한다면 기본 몇만원은 하는 돈이였는데 말이죠.ㅎㅎ

아무튼 이렇게 글을 마치겠습니다.