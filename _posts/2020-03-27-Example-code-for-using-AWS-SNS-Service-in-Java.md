---
layout: post
title: Example code for using AWS SNS Service in Java
date: 2020-03-27
Author: Mufu Han
toc: true
tags: [Java, AWS, Gradle]
--- 
This is a Spring Boot application and I use Gradle to maintain dependencies. I will show you a example code which implements a simple SNS service to send SMS texts to users' phone. 

<!-- more -->

## Add dependencies in build.gradle

My build.gradle looks like:

````
plugins {
	id 'org.springframework.boot' version '2.2.5.RELEASE'
	id 'io.spring.dependency-management' version '1.0.9.RELEASE'
	id 'java'
}

group = 'io.github.hanmufu'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '1.8'

repositories {
	mavenCentral()
}

dependencies {
    compile (
            'com.amazonaws:aws-lambda-java-core:1.1.0',
            'com.amazonaws:aws-lambda-java-events:1.1.0',
            'com.amazonaws:aws-java-sdk:1.11.255', 
						'com.alibaba:fastjson:1.2.54'
    )
	  implementation 'org.springframework.boot:spring-boot-starter-data-mongodb'
		implementation 'org.springframework.boot:spring-boot-starter-web'
		testImplementation('org.springframework.boot:spring-boot-starter-test') {
		exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
	}
}

test {
	useJUnitPlatform()
}
````

## Implement it with Controller and Service layers

I have a SMSController, SMSService and a SMSServiceImpl. 

SMSController is used to receive HTTP requests from clients. It uses RequestMapping to map web requests onto assigned handler methods. In the handler method, It will call SMSServices to achieve the sending texts task. SMSService is a interface though, we have to implement it in SMSServiceImpl. The reason we need a interface between Controller and Service is to decoupling them. With a interface, the implementation in Service layer is totally transparent for Controller. Also, it is good for parallel development. 

SMSController.java:

```java
package io.github.hanmufu.IPMap.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;
import io.github.hanmufu.IPMap.service.SMSService;

import java.util.*;

@RestController
@RequestMapping(value = "/SMS")
public class SMSController {

    @Autowired
    private SMSService smsService;

    @RequestMapping(value = "/send", method = RequestMethod.POST)
    public String sendSMS(@RequestBody Map<String, Object> params) {
        String phoneNumber = params.get("phoneNumber").toString();
        String message = params.get("message").toString();
        String areaCode = params.get("areaCode").toString();
        return smsService.sendSMS(phoneNumber, message, areaCode);
    }

}
```

SMSService.java:

```java
package io.github.hanmufu.IPMap.service;

public interface SMSService {

    String sendSMS(String phoneNumber, String message, String areaCode);

}
```

SMSServiceImpl.java:

```java
package io.github.hanmufu.IPMap.serviceImpl;

import com.alibaba.fastjson.JSONObject;
import io.github.hanmufu.IPMap.service.SMSService;
import org.springframework.stereotype.Service;

import java.util.HashMap;
import java.util.Map;
import java.util.regex.Pattern;

import com.amazonaws.auth.BasicAWSCredentials;
import com.amazonaws.services.sns.AmazonSNSClient;
import com.amazonaws.services.sns.model.MessageAttributeValue;
import com.amazonaws.services.sns.model.*;

@Service
public class SMSServiceImpl implements SMSService {

    public String sendSMS(String phoneNumber, String message, String areaCode) {
        JSONObject responseData = new JSONObject();
        if(areaCode.equals("+1") && phoneNumber.length() != 10) {
            responseData.put("alert", "phone number should be 10 digits");
            return responseData.toJSONString();
        }else if(areaCode.equals("+86") && phoneNumber.length() != 11) {
            responseData.put("alert", "phone number should be 11 digits");
            return responseData.toJSONString();
        }
        Pattern pattern = Pattern.compile("[0-9]*");
        if(!pattern.matcher(phoneNumber).matches()) {
            responseData.put("alert", "not a phone number");
            return responseData.toJSONString();
        }
        BasicAWSCredentials awsCreds = new BasicAWSCredentials("YOUR accessKey", "YOUR secretKey");
        AmazonSNSClient snsClient = new AmazonSNSClient(awsCreds);
        message = "From Mufu Han: " + message;
        phoneNumber = areaCode + phoneNumber;
        System.out.println("text message is: " + message);
        System.out.println("target phone number is: " + phoneNumber);
        Map<String, MessageAttributeValue> smsAttributes = new HashMap<String, MessageAttributeValue>();
        smsAttributes.put("AWS.SNS.SMS.SenderID", new MessageAttributeValue()
                .withStringValue("MufuHan") //The sender ID shown on the device.
                .withDataType("String"));
        smsAttributes.put("AWS.SNS.SMS.MaxPrice", new MessageAttributeValue()
                .withStringValue("0.5") //Sets the max price
                .withDataType("Number"));
        smsAttributes.put("AWS.SNS.SMS.SMSType", new MessageAttributeValue()
                .withStringValue("Promotional") //Sets the type to promotional.
                .withDataType("String"));
        sendSMSMessage(snsClient, message, phoneNumber, smsAttributes);
        responseData.put("alert", "Sent successfully");
        return responseData.toJSONString();
    }

    public static void sendSMSMessage(AmazonSNSClient snsClient, String message,
                                      String phoneNumber, Map<String, MessageAttributeValue> smsAttributes) {
        PublishResult result = snsClient.publish(new PublishRequest()
                .withMessage(message)
                .withPhoneNumber(phoneNumber)
                .withMessageAttributes(smsAttributes));
        System.out.println("text sent, message ID: " + result); // Prints the message ID.
    }
}
```

