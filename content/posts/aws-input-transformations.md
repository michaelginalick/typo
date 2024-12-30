---
title: 'AWS Input Transformations'
date: 2024-12-26T17:30:58-06:00
summary: "Using a simple AWS tool to improve local and cloud development workflows"
description: "Input transformers for more structured lambda development"
toc: false
readTime: true
autonumber: true
math: false
tags: ["Go", "AWS lambda", "input transformers"]
showTags: true
hideBackToTop: false
---

For the past few years, I've been working primarily in the AWS environment, specifically with AWS API Gateway, EventBridge, Lambda, and SQS. I won’t delve into SQS since a queue is a queue, and I don’t have much to offer in terms of advice—other than recommending processing in batches instead of one item at a time and enabling long polling. These two strategies will get you pretty far based on my experience.

This post, however, is about a feature that, in my opinion, is essential for working with EventBridge and Lambda: [Amazon EventBridge Input Transformations](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-transform-target-input.html).

# Why Use Input Transformers?

I won’t go into the details of what an input transformer is or how to use it—the AWS documentation does a great job of explaining that. Instead, I’ll focus on why you should use them.

When an event is consumed by a Lambda function, the `handler` function signature, without a transformer, might look like this:

```go
func handler(ctx context.Context, event map[string]interface{}) error {
	eventJSON, _ := json.MarshalIndent(event, "", "  ")
	fmt.Printf("Event received: %s\n", string(eventJSON))
	return nil
}

func main() {
	lambda.Start(handler)
}
```

The event is a map[string]interface{}. This doesn’t provide much information about the structure of the event being consumed. You could improve it slightly by defining a type for the event:

```golang
type EventBridgeEvent struct {
	Version    string                 `json:"version"`
	ID         string                 `json:"id"`
	DetailType string                 `json:"detail-type"`
	Source     string                 `json:"source"`
	Account    string                 `json:"account"`
	Time       string                 `json:"time"`
	Region     string                 `json:"region"`
	Resources  []string               `json:"resources"`
	Detail     map[string]interface{} `json:"detail"`
}

func handler(ctx context.Context, event EventBridgeEvent) error {
	fmt.Printf("Event received: %+v\n", event)
	return nil
}

func main() {
	lambda.Start(handler)
}
```

This is better, but notice how the Detail field is still a map[string]interface{}? In my experience, the Detail field typically contains the data that the Lambda logic relies on. This is where input transformers come into play—they allow you to take the raw event data and map it into something more structured and useful.

# Example Use Case

Let’s say you have an event that fires when a database row is updated. The raw event might look like this:

```json
{
  "version": "14",
  "id": "7bf73129-1428-4cd3-a780-95db273d1602",
  "detail-type": "DB Address change Notification",
  "source": "aws.ec2",
  "account": "123456789012",
  "time": "2015-11-11T21:29:54Z",
  "region": "us-east-1",
  "resources": [
    "arn:aws:ec2:us-east-1:123456789012:instance/i-abcd1111"
  ],
  "detail": {
    "action": "address changed",
    "user_id": "112234321",
  }
}
```

Here, the Lambda function only cares about two fields: the action ("address changed") and the user_id. By using an input transformer, you can simplify the payload, sending only the relevant data to the Lambda in a predefined structure:

```json
{
  "data": {
    "action": "address changed",
    "user_id": "11nbd2a2s3eq4321"
  }
}

```

And the updated function signature can now look like this:

```golang
func handler(ctx context.Context, event json.RawMessage) error {
	fmt.Printf("Event received: %s\n", string(event))
	return nil
}

func main() {
	lambda.Start(handler)
}

```

This may not seem like a significant change, but now you can define and validate the event structure within your application. For example:

```golang
type Action struct {
	Action  string `json:"action"`
	UserID  string `json:"user_id"`
  // Other fields the action may need
}


// implements the unmarshal interface for JSON conversions
func (a *Action) UnmarshalAction(action json.RawMessage) error {
  // Validate the action here
  // Build out the action based on the data provided
}


func handler(ctx context.Context, event json.RawMessage) error {
  var action Action
  if err := action.UnmarshalAction(event); err != nil {
		return err
	}

  // do something with the action
	return nil
}

func main() {
	lambda.Start(handler)
}

```

# Benefits of This Approach

## Improved Readability and Maintainability
With this small change, your codebase becomes cleaner. The payload is immediately validated and converted into a structured type, making it easier to work with than a map[string]interface{}.

## Enhanced Testing and Local Development
Testing becomes much easier when working with structured JSON payloads. Instead of defining complex map[string]interface{} objects, you can use simple JSON files for testing.

Here’s an example folder structure:

```bash
cmd/
  dev/
    test.json
    sample1.json
    sample2.json
    main.go
```
Each JSON file represents a test payload. You can edit these files easily to simulate different scenarios, both happy and unhappy paths.

## Simplified Cloud Testing

The same JSON payloads used locally can also be used in the AWS console for testing. This consistency makes debugging and validation much more straightforward.

# Conclusion

Input transformers are a powerful tool for simplifying cloud and local workflows. They require minimal effort to set up and save significant time moving forward. With [Terraform](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_event_target) support, you only need to define them once.

In my team, we use this pattern in all the Lambda functions we maintain. It has made the codebase easier to maintain and has significantly improved the onboarding experience for new developers. Input transformers are a tool I highly recommend for anyone working with AWS Lambda.
