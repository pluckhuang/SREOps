1. ec2 host, ip, storage 都是按小时$0.00x 收费
   ![alt text](imgs/image1.png)
2. s3, 上传与下载要授权相关权限
3. ec2 有系统故障和实例故障监控,  结合 cloudwatch 可于实例故障时自动重启, cloudwatch 触发时可结合lambda 操作, 触发告警 比如发消息到slack.