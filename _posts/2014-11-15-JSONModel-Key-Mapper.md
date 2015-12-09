---
title: Customize JSONModel Key Mapping Rules
date: 2014-11-15 16:00
layout: post
category: projects
comments: true
tags: iOS JSONModel objc_runtime kvc
description: JSONModel is an iOS library, which allows rapid creation of smart data models. It can be used in iOS or OSX apps. JSONModel automatically introspects your model classes and the structure of your JSON input and reduces drastically the amount of code you have to write.
---

JSONModel is an iOS library, which allows rapid creation of smart data models. It can be used in iOS or OSX apps. JSONModel automatically introspects your model classes and the structure of your JSON input and reduces drastically the amount of code you have to write.

Upfront was the descripton from JSONModel Official (Github)[https://github.com/icanzilb/JSONModel] page. In plain words (codes), if we have got such json input:

    @{@“id”: @“123456”, @“name”: @“little”, @“model”: @{@“no”: @“model 1586”,  @“desc”: @“a short description”}}

It can be directly mapped into the following class:

    @interface Product : JSONModel
    @property (copy, nonatomic) NSString *id;
    @property (copy, nonatomic) NSString *name;
    @property (copy, nonatomic) NSString *modelNo;
    @property (copy, nonatomic) NSString *modelDesc;
    @end

by overriding its `keyMapper` class function:
    
    +(JSONKeyMapper*)keyMapper
    {
        return [[JSONKeyMapper alloc] initWithDictionary:@{
        @“model.no”: @“modelNo”,
        @“model.desc”: @“modelDesc”
        }];
    }

Specifying `model.no` rule in json format is mapped into `modelNo` property in the objc object. By setting these rule, we will get an `Product` object will its properties set to coresponding json inputs exactly.

However, by the time we have the reverse requirement, which is mapping `modelNo` key in json to `model.no` property in objc object, the JSONModel fails to do the mapping rule. That means, if we have such json input:

    {@“id”: @“123456”, @“name”: @“little”, @“modelNo”: @“model 1586”,  @“modelDesc”: @“a short description”}

And the definition of JSONModel subclass:

    @interface Model : JSONModel
    @property (copy, nonatomic) NSString * no;
    @property (copy, nonatomic) NSString * desc;
    @end

    @interface Product : JSONModel
    @property (copy, nonatomic) NSString * id;
    @property (copy, nonatomic) NSString * name;
    @property (strong, nonatomic) Model * model;
    @end

And overriding `keyMapper`:

    +(JSONKeyMapper*)keyMapper
    {
      return [[JSONKeyMapper alloc] initWithDictionary:@{
        @“modelNo”: @“model.no”,
        @“modelDesc”: @“model.desc”
      }];
    }

We will not set the json values to our `Product` object as expected. What we will do is to make that happen.

<br />

