---
title: 开闭原则
date: 2021-02-08 14:17:10
tags:
  - programming
---

## 什么是开闭原则

**需求变化** 是让开发者最为头痛的问题之一。通常在新增一个功能的时候，都会伴随着修改原有的代码。因此，理想情况是在新增功能时尽可能减少对已有代码的改动，避免因改动代码引入新的问题。

由此产生了一个编码设计原则: 开闭原则 (Open Close Principle)。

> Software entities like classes, modules and functions should be **open for extension** but **closed for modifications**.

## 举个栗子 —— 用户注册模块

### 原始需求

业务初期，只允许使用邮箱注册，因此，需要实现向邮箱发送验证码的功能。

定义邮箱发送类，用于发送验证码:

``` go
type EmailMessageSender struct {
}

func (e *EmailMessageSender) SendVerificationCode(code string) {
  fmt.Println("Code: ", code)
}
```

定义服务类，调用邮箱发送类发送消息:

``` go
type MessageService struct {
  emailSender *EmailMessageSender
}

func NewMessageService() *MessageService {
  return &MessageService{
    emailSender: &EmailMessageSender{}
  }
}

func (s *MessageService) SendVerificationCode(code string) {
  s.emailSender.SendVerificationCode(code)
}
```

调用方式为:

``` go
func main() {
  service := NewMessageService()
  service.SendVerificationCode("10086")
}
```

### 增加需求

后来业务要求，允许使用手机号注册，通过短信的方式接受验证码。

因此，增加发送短信类:

``` go
type ShortMessageSender struct {
}

func (e *ShortMessageSender) SendVerificationCode(code string) {
  fmt.Println("Code: ", code)
}
```

同时，需要修改 `MessageService` 类:

``` go
type MessageService struct {
  emailSender        *EmailMessageSender
  shortMessageSender *ShortMessageSender
}
```

还需要通过某种方式制定使用哪种途径发送消息，例如增加一个表示类型的参数:

``` go
type MessageSender int

const (
  MessageSenderEmail        MessageSender = iota
  MessageSenderShortMessage
)

func NewMessageService(typ MessageSender) *MessageService {
  switch typ {
    case MessageSenderEmail:
      return &MessageService{
        emailSender: &EmailMessageSender{}
      }
    case MessageSenderShortMessage:
      return &MessageService{
        shortMessageSender: &ShortMessageSender{}
      }
    default:
      panic(fmt.Sprintf("unknown sender type: %s", typ))
  }
}

func (s *MessageService) SendVerificationCode(code string) {
  if s.emailSender != nil {
    s.emailSender.SendVerificationCode(code)
  }
  if s.shortMessageSender != nil {
    s.shortMessageSender.SendVerificationCode(code)
  }
}
```

调用方式为:

``` go
func main() {
  {
    service := NewMessageService(MessageSenderEmail)
    service.SendVerificationCode("10086")
  }
  {
    service := NewMessageService(MessageSenderShortMessage)
    service.SendVerificationCode("10086")
  }
}
```

### 小结

上面的实现方式违背了 `OCP`: 在增加新类型 `ShortMessageSender` 的同时，为了能使用这个类，需要同时修改函数 `NewMessageService` 与函数 `MessageService::SendVerificationCode`，手动确定使用哪种方式发送消息。

并且可以确定以后每增加一种发送消息的类型，都需要同时修改这两个函数。

### 重构

首先，可以观察到，无论是通过类型 `EmailMessageSender` 发送消息，亦或是通过类型 `ShortMessageSender` 发送消息，二者对于 `MessageService` 都只是发送消息的一种实现方式，而 `MessageService` 并不关心具体使用的方式是什么。由此可以考虑将这个功能抽象为一个接口:

``` go
type Sender interface {
  SendVerificationCode(code string)
}
```

`MessageService` 只需要持有这个接口即可:

``` go
type MessageService struct {
  sender Sender
}

func NewMessageService(sender Sender) *MessageService {
  return &MessageService{
    sender: sender,
  }
}

type (s *MessageService) SendVerificationCode(code string) {
  s.sender.SendVerificationCode(code)
}
```

接下来就是水到渠成的事情，让 `EmailMessageSender` 与 `ShortMessageSender` 分别实现接口 `Sender`:

``` go
type EmailMessageSender struct {
}

func (e *EmailMessageSender) SendVerificationCode(code string) {
  fmt.Println("Code: ", code)
}

type ShortMessageSender struct {
}

func (e *ShortMessageSender) SendVerificationCode(code string) {
  fmt.Println("Code: ", code)
}
```

最后，调用方式为:

``` go
func main() {
  {
    service := NewMessageService(&EmailMessageSender{})
    service.SendVerificationCode("10086")
  }
  {
    service := NewMessageService(&ShortMessageSender{})
    service.SendVerificationCode("10086")
  }
}
```

使用这种方式，若要增加新的发送方式，只需要增加对应的类型，并使其实现接口 `Sender` 即可，而无需修改除 `main` 以外的其他函数。达到 **open for extension** but **closed for modifications**。

## 总结

**开闭原则** 的目标是指导如何 **提高代码可扩展性**，因此是众多设计模式主要遵从的设计原则。

熟练使用这项原则，需要开发者具备扩展意识、抽象意识、封装意识等。在写代码之前，要认真思考，未来的需求可能会改变哪里。将可能的变化进行抽象，对外提供不变的接口。
