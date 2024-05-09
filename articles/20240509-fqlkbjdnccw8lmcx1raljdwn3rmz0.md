---
title: "StepFunctionsからECS Taskを実行してコケたらSlack通知したい"
emoji: "😊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS"]
published: true
---

cronジョブやone shotのジョブを実行する環境としてECS Taskを利用しています。  
その際、リトライやタイムアウト、同時実行制御のことを考えるのが面倒なのでStepFunctionsで包むのはよくある使用法だと思います。  
で、StepFunctionsがコケてエラーになったときはSlackに通知したい。  
これもアルアルかなと。  
ただ、このためにSlackでbotトークンを発行してLambdaを作って、とか面倒でやりたくない！  
という強い気持ちでSNS + AWS Chatbot経由での通知を構築しました。  
Terraformを使用します。

## StepFunctions

ECSクラスタ名によって、ECSタスクのURLは変わります。  
以下はsampleという名前のECSクラスタをap-northeast-1に立てた場合の例です。

```hcl
locals {
  ecs_task_url       = "https://ap-northeast-1.console.aws.amazon.com/ecs/v2/clusters/sample/tasks/"
}

resource "aws_sfn_state_machine" "sample" {
  definition = jsonencode(
    {
      StartAt = "RunTask"
      States = {
        RunTask = {
          Type     = "Task"
          Resource = "arn:aws:states:::ecs:runTask.sync"
          Parameters = {
            LaunchType     = "FARGATE"
            Cluster        = aws_ecs_cluster.sample.arn
            TaskDefinition = replace(aws_ecs_task_definition.sample.arn, "/:\\d+$/", "")
            Overrides = {
              ContainerOverrides = [
                {
                  Name        = "sample"
                  "Command.$" = "$.command"
                }
              ]
            }
            NetworkConfiguration = {
              AwsvpcConfiguration = {
                AssignPublicIp = "ENABLED"
                SecurityGroups = [aws_security_group.sample.id]
                Subnets        = [aws_subnet.sample.id]
              }
            }
          }
          End = true
          Catch = [
            {
              ErrorEquals = [
                "States.ALL"
              ]
              Next = "ModifyOutput1"
            }
          ]
        }
        ModifyOutput1 = {
          Type = "Pass"
          Parameters = {
            "Cause.$" = "States.StringToJson($.Cause)"
          }
          Next = "ModifyOutput2"
        }
        ModifyOutput2 = {
          Type = "Pass"
          Parameters = {
            "Cause.$" = "States.StringSplit($.Cause.TaskArn, '/')"
          }
          Next = "ModifyOutput3"
        }
        ModifyOutput3 = {
          Type = "Pass"
          Parameters = {
            "Cause.$" = "States.Format('see ${local.ecs_task_url}{}/logs?region=ap-northeast-1', $.Cause[2])"
          }
          Next = "SNSPublish"
        }
        SNSPublish = {
          Type     = "Task"
          Resource = "arn:aws:states:::sns:publish"
          Parameters = {
            TopicArn = aws_sns_topic.sample.arn
            Message = {
              version = "1.0"
              source  = "custom"
              content = {
                "description.$" = "$.Cause"
                title           = "error has occurred"
                textType        = "client-markdown"
              }
            }
          }
          Next = "Fail"
        }
        Fail = {
          Type : "Fail"
        }
      }
    }
  )
  name     = "sample"
  role_arn = aws_iam_role.sample.arn
}
```

説明のため、Retry・Timeoutは設定していません。  
RunTaskが失敗した場合は、全部CatchしてSNS経由でAWS Chatbotを起動し、Slackへ通知します。

RunTask失敗時のoutputは以下のような形式です。  
ここから、ECSタスクIDを取り出したいので、StepFunctionsの組込み数を使用します。

<!-- markdownlint-disable -->
```json
{
  "name": "RunTask",
  "output": "{\"Error\":\"States.TaskFailed\",\"Cause\":\"{\\\"Attachments\\\":[{\\\"Details\\\":[{\\\"Name\\\":\\\"subnetId\\\",\\\"Value\\\":\\\"subnet-00000000000000000\\\"},{\\\"Name\\\":\\\"networkInterfaceId\\\",\\\"Value\\\":\\\"eni-00000000000000000\\\"},{\\\"Name\\\":\\\"macAddress\\\",\\\"Value\\\":\\\"0a:0a:0a:0a:0a:0a\\\"},{\\\"Name\\\":\\\"privateIPv4Address\\\",\\\"Value\\\":\\\"10.0.0.1\\\"}],\\\"Id\\\":\\\"7975e065-b8b9-477d-b7f2-4ececfa1a5e3\\\",\\\"Status\\\":\\\"DELETED\\\",\\\"Type\\\":\\\"eni\\\"}],\\\"Attributes\\\":[{\\\"Name\\\":\\\"ecs.cpu-architecture\\\",\\\"Value\\\":\\\"arm64\\\"}],\\\"AvailabilityZone\\\":\\\"ap-northeast-1c\\\",\\\"ClusterArn\\\":\\\"arn:aws:ecs:ap-northeast-1:000000000000:cluster/sample\\\",\\\"Connectivity\\\":\\\"CONNECTED\\\",\\\"ConnectivityAt\\\":1715151108682,\\\"Containers\\\":[{\\\"ContainerArn\\\":\\\"arn:aws:ecs:ap-northeast-1:000000000000:container/sample/7446f4e4ff8441eabc8919fe51d8cac5/f5d4dacc-0c3f-43b5-a12d-3b6faf9b0ce5\\\",\\\"Cpu\\\":\\\"0\\\",\\\"ExitCode\\\":1,\\\"GpuIds\\\":[],\\\"Image\\\":\\\"000000000000.dkr.ecr.ap-northeast-1.amazonaws.com/sample:latest\\\",\\\"ImageDigest\\\":\\\"sha256:478663e9244a6f8c1866ea8e59fdc3e8356e08cebb99e5fcc92b2d8dacf891b8\\\",\\\"LastStatus\\\":\\\"STOPPED\\\",\\\"ManagedAgents\\\":[],\\\"Name\\\":\\\"sample\\\",\\\"NetworkBindings\\\":[],\\\"NetworkInterfaces\\\":[{\\\"AttachmentId\\\":\\\"7975e065-b8b9-477d-b7f2-4ececfa1a5e3\\\",\\\"PrivateIpv4Address\\\":\\\"10.0.0.1\\\"}],\\\"RuntimeId\\\":\\\"7446f4e4ff8441eabc8919fe51d8cac5-3286335836\\\",\\\"TaskArn\\\":\\\"arn:aws:ecs:ap-northeast-1:000000000000:task/sample/7446f4e4ff8441eabc8919fe51d8cac5\\\"}],\\\"Cpu\\\":\\\"256\\\",\\\"CreatedAt\\\":1715151105029,\\\"DesiredStatus\\\":\\\"STOPPED\\\",\\\"EnableExecuteCommand\\\":false,\\\"EphemeralStorage\\\":{\\\"SizeInGiB\\\":20},\\\"ExecutionStoppedAt\\\":1715151119396,\\\"Group\\\":\\\"family:sample\\\",\\\"InferenceAccelerators\\\":[],\\\"LastStatus\\\":\\\"STOPPED\\\",\\\"LaunchType\\\":\\\"FARGATE\\\",\\\"Memory\\\":\\\"512\\\",\\\"Overrides\\\":{\\\"ContainerOverrides\\\":[{\\\"Command\\\":[],\\\"Environment\\\":[],\\\"EnvironmentFiles\\\":[],\\\"Name\\\":\\\"sample\\\",\\\"ResourceRequirements\\\":[]}],\\\"InferenceAcceleratorOverrides\\\":[]},\\\"PlatformVersion\\\":\\\"1.4.0\\\",\\\"PullStartedAt\\\":1715151117893,\\\"PullStoppedAt\\\":1715151119256,\\\"StartedAt\\\":1715151119431,\\\"StartedBy\\\":\\\"AWS Step Functions\\\",\\\"StopCode\\\":\\\"EssentialContainerExited\\\",\\\"StoppedAt\\\":1715151144686,\\\"StoppedReason\\\":\\\"Essential container in task exited\\\",\\\"StoppingAt\\\":1715151129486,\\\"Tags\\\":[],\\\"TaskArn\\\":\\\"arn:aws:ecs:ap-northeast-1:000000000000:task/sample/7446f4e4ff8441eabc8919fe51d8cac5\\\",\\\"TaskDefinitionArn\\\":\\\"arn:aws:ecs:ap-northeast-1:000000000000:task-definition/sample:6\\\",\\\"Version\\\":5}\"}",
  "outputDetails": {
    "truncated": false
  }
}
```
<!-- markdownlint-enable-->

## StepFunctions組込み数

組込み数をネストして使う、みたいなことができればよかったのですが、できなそうでした。  
(StepFunctionsコンソールで保存しようとすると、バリデーションで落ちる)  
諦めて複数ステップで変換します。

### 1. escape済みJSON文字列をJSONに変換する

```hcl
   ModifyOutput1 = {
       Type = "Pass"
       Parameters = {
           "Cause.$" = "States.StringToJson($.Cause)"
       }
       Next = "ModifyOutput2"
   }
```

### 2. JSONからTaskArnを取り出し、'/'を区切り文字として配列に変換する

```hcl
   ModifyOutput2 = {
       Type = "Pass"
       Parameters = {
           "Cause.$" = "States.StringSplit($.Cause.TaskArn, '/')"
       }
       Next = "ModifyOutput3"
   }
```

```json
{
  "Cause": "arn:aws:ecs:ap-northeast-1:000000000000:task/sample/7446f4e4ff8441eabc8919fe51d8cac5"
}
```

↓

```json
{
  "Cause": ["arn:aws:ecs:ap-northeast-1:000000000000:task", "sample", "7446f4e4ff8441eabc8919fe51d8cac5"]
}
```

### 3. 配列からタスクID部分を取り出し、ECSタスクのログを表示するURLに変換する

```hcl
   ModifyOutput3 = {
       Type = "Pass"
       Parameters = {
           "Cause.$" = "States.Format('see ${local.ecs_task_url}{}/logs?region=ap-northeast-1', $.Cause[2])"
       }
       Next = "SNSPublish"
   }
```

```json
{
  "Cause": ["arn:aws:ecs:ap-northeast-1:000000000000:task", "sample", "7446f4e4ff8441eabc8919fe51d8cac5"]
}
```

↓

```json
{
  "Cause": "https://ap-northeast-1.console.aws.amazon.com/ecs/v2/clusters/sample/tasks/7446f4e4ff8441eabc8919fe51d8cac5/logs?region=ap-northeast-1"
}
```

### 4. SNSに渡して、AWS Chatbotのカスタム通知用に整形する

```hcl
   SNSPublish = {
       Type     = "Task"
       Resource = "arn:aws:states:::sns:publish"
       Parameters = {
           TopicArn = aws_sns_topic.sample.arn
           Message = {
               version = "1.0"
               source  = "custom"
               content = {
                   "description.$" = "$.Cause"
                   title           = "error has occurred"
                   textType        = "client-markdown"
               }
           }
       }
       Next = "Fail"
   }
```

## まとめ

CloudWatch LogsのURLを飛ばすことも考えたのですが、いったんは簡単に作れそうだったECSタスクのURLとしています。  
ということで、無事にLambda不要でSlack通知できるようになりました。
