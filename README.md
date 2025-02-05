# ebs_autoexpend

resource "aws_ssm_parameter" "cloudwatch_agent_config" {
  name  = "/AmazonCloudWatchAgent/config"
  type  = "String"
  value = jsonencode({
    agent = {
      metrics_collection_interval = 60
    }
    metrics = {
      namespace = "Custom/EBSMonitoring"
      append_dimensions = {
        InstanceId = "${aws_instance.ec2_instance.id}"
      }
      metrics_collected = {
        disk = {
          measurement = ["used_percent"]
          resources   = ["*"]
          ignore_file_system_types = ["sysfs", "devtmpfs"]
        }
      }
    }
  })
}


resource "aws_ssm_document" "install_cw_agent" {
  name          = "InstallCloudWatchAgent"
  document_type = "Command"

  content = <<DOC
  {
    "schemaVersion": "2.2",
    "description": "Install and configure CloudWatch Agent",
    "mainSteps": [
      {
        "action": "aws:runShellScript",
        "name": "InstallCloudWatchAgent",
        "inputs": {
          "runCommand": [
            "sudo yum install -y amazon-cloudwatch-agent",
            "sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c ssm:/AmazonCloudWatchAgent/config -s"
          ]
        }
      }
    ]
  }
  DOC
}


resource "aws_cloudwatch_metric_alarm" "disk_usage_alarm" {
  alarm_name          = "HighDiskUsageAlarm"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 1
  metric_name         = "used_percent"
  namespace           = "Custom/EBSMonitoring"
  period              = 60
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "Triggers when disk usage exceeds 80%."
  alarm_actions       = [aws_sns_topic.alarm_notification.arn]
  dimensions = {
    InstanceId = "${aws_instance.ec2_instance.id}"
  }
}



resource "aws_sns_topic" "alarm_notification" {
  name = "EBSVolumeResizingTopic"
}



resource "aws_sns_topic_subscription" "sns_to_api_gateway" {
  topic_arn = aws_sns_topic.alarm_notification.arn
  protocol  = "https"
  endpoint  = aws_api_gateway_integration.api_gateway_invoke.invoke_url
}


resource "aws_api_gateway_rest_api" "ebs_alerts_api" {
  name        = "EBSVolumeResizingAPI"
  description = "API Gateway to trigger Lambda for EBS resizing"
}


resource "aws_api_gateway_resource" "alert_resource" {
  rest_api_id = aws_api_gateway_rest_api.ebs_alerts_api.id
  parent_id   = aws_api_gateway_rest_api.ebs_alerts_api.root_resource_id
  path_part   = "alert"
}

resource "aws_api_gateway_method" "post_alert" {
  rest_api_id   = aws_api_gateway_rest_api.ebs_alerts_api.id
  resource_id   = aws_api_gateway_resource.alert_resource.id
  http_method   = "POST"
  authorization = "NONE"
}


resource "aws_api_gateway_integration" "api_gateway_invoke" {
  rest_api_id             = aws_api_gateway_rest_api.ebs_alerts_api.id
  resource_id             = aws_api_gateway_resource.alert_resource.id
  http_method             = aws_api_gateway_method.post_alert.http_method
  integration_http_method = "POST"
  type                    = "AWS_PROXY"
  uri                     = aws_lambda_function.ebs_resizer.invoke_arn
}


resource "aws_api_gateway_deployment" "ebs_alerts_deployment" {
  rest_api_id = aws_api_gateway_rest_api.ebs_alerts_api.id
  stage_name  = "prod"
}
