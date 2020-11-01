---
layout: post
title: "Serverless Framework with AWS Lambda"
date: 2020-11-01 15:00:00 +0900
image: 12.jpg
tags: [aws, lambda]
categories: aws
---
{:refdef: style="text-align: center;"}
![lambda](/images/aws/serverless-framework/lambda.png)
{: refdef}

Amazon의 Lambda 는 serverless한 서비스를 제공합니다. 그리고 Lambda에서는 코드를 업로드하기만 하면, 높은 가용성으로 코드를 실행 및 확장하는 데 필요한 모든 것을 처리합니다. 다른 AWS 서비스에서 코드를 자동으로 트리거하도록 설정하거나 웹 또는 모바일 앱에서 직접 코드를 호출할 수 있습니다.

가격 면에서도 이점이 있는데요. 사용한 컴퓨팅 시간만큼만 비용을 지불하고, 코드가 실행되지 않을 때는 요금이 부과되지 않습니다.

---
### 그러나 패키지 관리 및 deploy가 힘들다.
기존 파이썬 같은 경우는 pip 명령어를 사용하여 패키지를 설치하였습니다.
{% highlight bash %}
$ pip install {service}
{% endhighlight %}
참으로 편리하죠. 딱 저 명령어만 입력을 하면 되니..

그런데 lambda에서는 패키지 설치 및 이용 면에서 힘든 점이 많았습니다.

보통 파이썬으로 개발하시는 분들은 가상환경을 많이 사용하실 텐데요.

virtualenv로 가상환경을 만드셨다면, lib/pythonX.X/site-packages/ 경로에는 pip 명령어로 설치한 패키지들이 저장됩니다.

만약 lambda에 코드를 upload할 시, A라는 패키지를 사용하고 싶으시다면 다음과 같은 방법을 이용해야 합니다.
{% highlight text %}
1. 업로드 할 폴더를 생성
2. lambda 코드 작성
3. pip install {A}
4. lib/pythonX.X/site-packages/에 해당하는 A폴더와 A-x.x.x.dist-info폴더를 1번에서 생성한 폴더에 복사
5. zip 파일로 압축 및 upload (위 그림의 upload 클릭하여 upload)
{% endhighlight %}
그런데 또 3, 4번의 방법이 통하지 않을 때가 있었습니다.

예를 들면, psycopg2(postgresql 패키지) 같은 경우는 여기 링크의 precompile된 것을 zip 파일로 넣어줘야 하더군요..(엄청나게 귀찮습니다.)

---

### Serverless Framework
이러한 번거로움을 피하기 위해 이미 많은 개발자 분들이 엄청난 삽질과 편리한 툴을 만들어 놓았습니다. ([apex][https://github.com/apex/apex], [zappa][https://github.com/Miserlou/Zappa], [chalice][https://github.com/aws/chalice])

그 중, 전 serverless framework를 이용하였습니다.

{:refdef: style="text-align: center;"}
![serverless](/images/aws/serverless-framework/serverless.png)
{: refdef}
{:refdef: style="text-align: center;"}
(설치 방법은 [여기](https://www.serverless.com/framework/docs/)를 참고하시면 됩니다.)
{: refdef}

저는 awscli와 access-key, secret-key를 컴퓨터에 설정을 하였는데요. 

이미 설정이 되어 있으신 분들도 계실 텐데 그렇지 않으시면, [여기](https://docs.aws.amazon.com/ko_kr/streams/latest/dev/kinesis-tutorial-cli-installation.html) 링크에서 보고 설치해주세요.
 
{:refdef: style="text-align: center;"}
![policy](/images/aws/serverless-framework/admin_policy.png)
{: refdef}
그리고 사용자 권한에는 개발용으로 위와 같은 권한을 주었습니다. 

(production 일때는 위와 같은 권한을 주지 마세요. 회사 정책에 따라, 개발자에 차등 권한을 주는게 원래 맞습니다.)

{% highlight bash %}
$ npm install -g serverless
{% endhighlight %}

npm으로 serverless를 설치합니다.

{% highlight bash %}
$ serverless create --template aws-python3 --name test --path test
{% endhighlight %}

serverless 설치가 완료되었다면, 위 명령어를 이용하여 serverless 프로젝트를 생성합니다.

template에는 개발하기 쉽게 미리 만들어 놓은 템플릿이 있습니다.

저는 개발환경은 python3이라, aws-python3를 이용하였습니다.

{% highlight bash %}
$ cd test
[jh.seo]~/test$ ls
handler.py     serverless.yml
{% endhighlight %}

test 폴더가 생성되고 그 폴더 안에는 두 가지 파일이 생성됩니다.

serverless.yml 파일에는 framework을 deploy하기 위한 여러 설정을 할 수 있습니다.

그리고 handler.py가 있습니다. 이 파일이 lambda에 deploy될 파이썬 코드입니다.

{% highlight bash %}
[jh.seo]~/test$ mv handler.py handler1.py
[jh.seo]~/test$ cp handler1.py handler2.py
{% endhighlight %}

실습을 위해 두 가지 파일을 생성합니다.

{% highlight python %}
import json
 
def hello1(event, context):
  print("handler1 hello")
{% endhighlight %}

handler1.py를 위와 같이 수정합니다. 메소드 명은 hello1으로 줍니다.

{% highlight python %}
import json
import numpy as np
   
def hello2(event, context):
  print("handler2 hello")
  test_np = np.arange(15).reshape(3, 5)
  
  print("numpy array: ")
  print(test_np)
{% endhighlight %}
그리고 handler2.py를 open하여 위와 같이 수정합니다.

numpy 패키지를 import 하고, 메소드 명은 hello2로 변경 뒤 numpy 어레이를 생성 및 출력하는 코드를 작성합니다.
{% highlight bash %}
[jh.seo]~$ virtualenv -p python3 testenv
[jh.seo]~$ source testenv/bin/activate
(testenv) [jh.seo]~/test$ pip install numpy
{% endhighlight %}

가상환경을 만들어, numpy 패키지를 설치합니다.

{% highlight bash %}
(testenv) [jh.seo]~/test$ pip freeze > requirements.txt
{% endhighlight %}

그리고 pip freeze 명령어를 이용하여 패키지가 설치된 목록을 requirements.txt 파일로 만듭니다

{% highlight bash %}
(testenv) [jh.seo]~/test$ npm init
(testenv) [jh.seo]~/test$ npm install --save serverless-python-requirements
{% endhighlight %}

그리고 npm 명령어를 이용하여 serverless-python-requirements를 설치합니다. 파이썬 패키지를 관리하기 위한 npm plugin입니다.

{% highlight yaml %}
service: test
 
provider:
  name: aws
  runtime: python3.6
  region: ap-northeast-2
  stage: dev
     
plugins:
  - serverless-python-requirements

custom:
  pythonRequirements:
    dockerizePip: true

functions:
  func1:
   handler: handler1.hello1
    
  func2:
    handler: handler2.hello2
{% endhighlight %}

serverless.yml 파일을 위와 같이 수정해줍니다.

9번-14번 라인은 위에서 설치한 serverless-python-requirements 플러그인을 사용하기 위해 필요한 부분입니다.

16번-21번라인은 handler1.py와 handler2.py의 hello1메소드, hello2메소드를 lambda의 handler로 사용하겠다는 뜻입니다.

{% highlight bash %}
(testenv) [jh.seo]~/test$ serverless deploy
Serverless: Installing required Python packages with python3.6...
Serverless: Docker Image: lambci/lambda:build-python3.6
Serverless: Linking required Python packages...
Serverless: Packaging service...
Serverless: Excluding development dependencies...
Serverless: Unlinking required Python packages...
Serverless: Creating Stack...
Serverless: Checking Stack create progress...
.....
Serverless: Stack create finished...
Serverless: Uploading CloudFormation file to S3...
Serverless: Uploading artifacts...
Serverless: Uploading service .zip file to S3 (22.92 MB)...
Serverless: Validating template...
Serverless: Updating Stack...
Serverless: Checking Stack update progress...
.........................
Serverless: Stack update finished...
Service Information
service: test
stage: dev
region: ap-northeast-2
stack: test-dev
api keys:
  None
endpoints:
  None
functions:
  func1: test-dev-func1
  func2: test-dev-func2
{% endhighlight %}

`serverless deploy` 명령어를 입력하여 deploy 합니다.
{:refdef: style="text-align: center;"}
![policy](/images/aws/serverless-framework/lambda-function.png)
{: refdef}
그러면 위와 같이 2가지 lambda가 생성이 됩니다.
{:refdef: style="text-align: center;"}
![policy](/images/aws/serverless-framework/lambda-cloudwatch.png)
{: refdef}
그리고 cloudwatch에 로그 스트림이 자동으로 생성이 됩니다.


{% highlight bash %}
(testenv) [jh.seo]~/test$ serverless invoke -f func1 --log
null
--------------------------------------------------------------------
START RequestId: 38dfd068-f45c-11e7-b2c8-49ca6b97f4d0 Version: $LATEST
handler1 hello
END RequestId: 38dfd068-f45c-11e7-b2c8-49ca6b97f4d0
REPORT RequestId: 38dfd068-f45c-11e7-b2c8-49ca6b97f4d0	Duration: 0.28 ms	Billed Duration: 100 ms 	Memory Size: 1024 MMax Memory Used: 22 MB	


(testenv) [jh.seo]~/test$ serverless invoke -f func2 --log
null
--------------------------------------------------------------------
START RequestId: 3bea7846-f45c-11e7-8898-fbe3b2f6fa1f Version: $LATEST
handler2 hello
numpy array: 
[[ 0  1  2  3  4]
 [ 5  6  7  8  9]
 [10 11 12 13 14]]
END RequestId: 3bea7846-f45c-11e7-8898-fbe3b2f6fa1f
REPORT RequestId: 3bea7846-f45c-11e7-8898-fbe3b2f6fa1f	Duration: 1.92 ms	Billed Duration: 100 ms 	Memory Size: 1024 MMax Memory Used: 54 MB
{% endhighlight %}

---
### Event 설정
개발을 하다 보면 lambda를 주기적(예를 들면 10분)으로 실행하고 싶다거나, s3에 object가 생성이 될 때 lambda를 실행하고 싶은 시나리오가 생길텐데요.

그럴 때는 아래와 같은 방법을 사용합니다.
#### Ex1) handler1.py의 hello1 메소드를 timeout을 3분으로 설정 및 10분 단위로 lambda를 실행시키고 싶을 때.
serverless.yml을 다음과 같이 수정합니다.

{% highlight yaml %}
functions:
  func1:
    handler: handler1.hello1
    timeout: 180
    events:
    - schedule: rate(10 minutes)
{% endhighlight %}

#### Ex2) s3에 특정 prefix를 가지는 object가 생성이 될 때 handler1.py의 hello1 메소드를 실행시키고 싶을 때.
{% highlight bash %}
functions:
  func1:
    handler: handler1.hello1
    timeout: 180
    events:
    - schedule: rate(10 minutes)
{% endhighlight %}
이미 s3 bucket이 존재한다면 위 명령어를 입력하여, serverless-external-s3-event 플러그인을 설치합니다.

{% highlight yaml %}
provider:
  name: aws
  runtime: python3.6
  region: ap-northeast-2
  stage: dev
  iamRoleStatements:
    -   Effect: "Allow"
        Action:
          - "s3:PutBucketNotification"
        Resource:
          Fn::Join:
            - ""
            - "arn:aws:s3:::{BUCKET NAME}"
plugins:
  - serverless-external-s3-event
  - serverless-python-requirements
{% endhighlight %}

그리고 serverless.yml 파일을 위와 같이 수정합니다. BUCKET NAME에는 s3의 bucket name을 넣어주시면 됩니다. 그리고 plugins에 serverless-external-s3-event를 추가합니다.

{% highlight yaml %}
functions:
  func1:
    handler: handler1.hello1
    events:
      - existingS3:
        bucket: {BUCKET_NAME}
        events: 
          - s3:ObjectCreated:*
        rules:
          - prefix: {prefix}
{% endhighlight %}

function 부분은 위와 같이 수정합니다. {BUCKET NAME}에는 마찬가지로 s3의 bucket name을 입력하시면 되구요. {prefix}에는 이벤트를 발생시키고자 하는 object의 prefix를 입력합니다. 예를 들면 s3 object에 key가 aa로 시작한다면 aa를 입력하시면 됩니다.

---
### 마무리.

지금까지 serverless framework를 이용한 lambda deploy 과정에 대해 살펴보았는데요. 

전 엄청 삽질을 하였지만.. 막상 해보시면 별꺼 아니라는 걸 아실꺼에요. ㅎㅎ

오늘도 즐거운 코딩 되시길!!