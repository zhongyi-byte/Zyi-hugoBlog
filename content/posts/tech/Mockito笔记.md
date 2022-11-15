---
title: "Mockito笔记"
date: 2022-11-15T17:53:37+08:00
lastmod: 2022-11-15T17:53:37+08:00
author: ["Zyi"]
keywords: 
- 
categories: 
- 
tags: 
- 
description: ""
weight:
slug: ""
draft: false # 是否为草稿
comments: true
reward: false # 打赏
mermaid: true #是否开启mermaid
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示路径
cover:
    image: "" #图片路径例如：posts/tech/123/123.png
    caption: "" #图片底部描述
    alt: ""
    relative: false
---

**Mockito学习笔记**

### 概要

Mockito是一个常见用于mock依赖对象的测试工具，本文重点讲解Mockito的基本使用方法以及实现原理。

### 使用方法

        @Test
        public void testMockito() {
            //mock 一个对象
            ArrayList<String> urlList = Mockito.mock(ArrayList.class);
            /**
             *打印mock对象名，可以看到是被代理的对象
             *java.util.ArrayList$$EnhancerByMockitoWithCGLIB$$c24a9da4
             */ 
            System.out.println(urlList.getClass().getName());
            // stub打桩
            when(urlList.size()).thenReturn(100);
            when(urlList.get(0)).thenReturn("abc");
            when(urlList.get(1)).thenReturn("def");
    
            String value0 = urlList.get(0); // abc
            String value1 = urlList.get(1); // def
            String value2 = urlList.get(2); // null
            int size = urlList.size();      // 100
          	// verify调用次数
            verify(urlList, times(2)).size(); //error
        }

常用的功能主要是stub打桩和verify验证。

stub用于对mock对象设置的方法和参数，返回给定的数据。

verify用于检查mock对象方法的执行情况。

### 实现原理

本文基于mockito 1.10.19版本

跟踪Mockito.mock方法，得到MockitoCore的mock方法

        public <T> T mock(Class<T> typeToMock, MockSettings settings) {
            if (!MockSettingsImpl.class.isInstance(settings)) {
                throw new IllegalArgumentException(
                        "Unexpected implementation of '" + settings.getClass().getCanonicalName() + "'\n"
                        + "At the moment, you cannot provide your own implementations that class.");
            }
            MockSettingsImpl impl = MockSettingsImpl.class.cast(settings);
            MockCreationSettings<T> creationSettings = impl.confirm(typeToMock);
            // 创建mock对象
          	T mock = mockUtil.createMock(creationSettings);
            mockingProgress.mockingStarted(mock, typeToMock);
            return mock;
        }

跟踪createMock方法，到MockUtil中的createMock：

        public <T> T createMock(MockCreationSettings<T> settings) {
          	// 创建handler
            MockHandler mockHandler = new MockHandlerFactory().create(settings);
    	
          	// 调用cglib创建mock对象
            T mock = mockMaker.createMock(settings, mockHandler);
    
            Object spiedInstance = settings.getSpiedInstance();
            if (spiedInstance != null) {
                new LenientCopyTool().copyToMock(spiedInstance, mock);
            }
    
            return mock;
        }

通过cglib创建mock对象：

        public <T> T createMock(MockCreationSettings<T> settings, MockHandler handler) {
            InternalMockHandler mockitoHandler = cast(handler);
            new AcrossJVMSerializationFeature().enableSerializationAcrossJVM(settings);
            return new ClassImposterizer(new InstantiatorProvider().getInstantiator(settings)).imposterise(
                    new MethodInterceptorFilter(mockitoHandler, settings), settings.getTypeToMock(), settings.getExtraInterfaces());
        }

Mockito使用cglib这个框架来生成代理类代码，然后用ClassLoader进行加载，将MethodInterceptorFilter作为代理类的拦截器。

    @Override
        public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy)
                throws Throwable {
            /**
             ....
             **/
            Invocation invocation = new InvocationImpl(proxy, mockitoMethod, args, SequenceNumber.next(), realMethod);
            // 最终调用InternalMockHandler.handle方法
            return handler.handle(invocation);
        }

### stub

stub可以指定方法返回值，我们跟踪when方法来看看它的实现。

MockitoCore.when方法

        public <T> OngoingStubbing<T> when(T methodCall) {
            // 标记stub开始
            mockingProgress.stubbingStarted();
            return (OngoingStubbing) stub();
        }
    
        public IOngoingStubbing stub() {
            IOngoingStubbing stubbing = mockingProgress.pullOngoingStubbing();
            if (stubbing == null) {
                mockingProgress.reset();
                reporter.missingMethodInvocation();
            }
            return stubbing;
        }

when方法就是返回上次mock方法调用封装好的OngoingStubbing。

### thenReturn

    // OngoingStubbingImpl   
    public OngoingStubbing<T> thenAnswer(Answer<?> answer) {
            if(!invocationContainerImpl.hasInvocationForPotentialStubbing()) {
                new Reporter().incorrectUseOfApi();
            }
    
            invocationContainerImpl.addAnswer(answer);
            return new ConsecutiveStubbing<T>(invocationContainerImpl);
        }

    // InvocationContainerImpl
    public void addAnswer(Answer answer, boolean isConsecutive) {
      	// 获取stub的调用
        Invocation invocation = invocationForStubbing.getInvocation();
        mockingProgress.stubbingCompleted(invocation);
        AnswersValidator answersValidator = new AnswersValidator();
        answersValidator.validate(answer, invocation);
    
        synchronized (stubbed) {
            if (isConsecutive) {
                // stubbed本质是一个LinkedList,这里把stub的
                // 调用（invocation）和返回（answer）绑定，如果匹配到调用，就返回绑定值
                stubbed.getFirst().addAnswer(answer);
            } else {
                stubbed.addFirst(new StubbedInvocationMatcher(invocationForStubbing, answer));
            }
        }
    }

如何匹配调用方法呢？通过handle方法，将调用与stubbed链表进行匹配查询，如果有相同的调用，就返回该绑定值

    public StubbedInvocationMatcher findAnswerFor(Invocation invocation) {
        synchronized (stubbed) {
          	// 查询匹配的调用
            for (StubbedInvocationMatcher s : stubbed) {
                if (s.matches(invocation)) {
                    s.markStubUsed(invocation);
                    invocation.markStubbed(new StubInfoImpl(s));
                    return s;
                }
            }
        }
        return null;
    }

### verify

MockitoCore.verify

    public <T> T verify(T mock, VerificationMode mode) {
        if (mock == null) {
            reporter.nullPassedToVerify();
        } else if (!mockUtil.isMock(mock)) {
            reporter.notAMockPassedToVerify(mock.getClass());
        }
        mockingProgress.verificationStarted(new MockAwareVerificationMode(mock, mode));
        return mock;
    }

mockito提供了多种VerificationMode,这里使用的是Times，实现了VerificationMode类，用于统计方法执行次数。

    // Times
    public void verify(VerificationData data) {
        if (wantedCount > 0) {
            MissingInvocationChecker missingInvocation = new MissingInvocationChecker();
            missingInvocation.check(data.getAllInvocations(), data.getWanted());
        }
        NumberOfInvocationsChecker numberOfInvocations = new NumberOfInvocationsChecker();
        numberOfInvocations.check(data.getAllInvocations(), data.getWanted(), wantedCount);
    }
    
    // NumberOfInvocationChecker
    public void check(List<Invocation> invocations, InvocationMatcher wanted, int wantedCount) {
            List<Invocation> actualInvocations = finder.findInvocations(invocations, wanted);
            
            int actualCount = actualInvocations.size();
            if (wantedCount > actualCount) {
                Location lastInvocation = finder.getLastLocation(actualInvocations);
                reporter.tooLittleActualInvocations(new Discrepancy(wantedCount, actualCount), wanted, lastInvocation);
            } else if (wantedCount == 0 && actualCount > 0) {
                Location firstUndesired = actualInvocations.get(wantedCount).getLocation();
                reporter.neverWantedButInvoked(wanted, firstUndesired); 
            } else if (wantedCount < actualCount) {
                Location firstUndesired = actualInvocations.get(wantedCount).getLocation();
                reporter.tooManyActualInvocations(wantedCount, actualCount, wanted, firstUndesired);
            }
            
            invocationMarker.markVerified(actualInvocations, wanted);
        }

如果判断失败，则抛出异常


