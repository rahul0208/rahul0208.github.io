---
layout: post
title: Testing Async Spring methods
#subtitle: Each post also has a subtitle
#image: /img/hello_world.jpeg
tags: [Unit Testing, Async, CountDownLatch]
---
Recently I was part of team which started a green field project. Our application was a Spring Boot application. We were strictly adhering to TestDrivenDevelopemnt. Spring has good testing support. We annotated our test classes with `SpringBootTest` and it worked for most of our `RestControllers` and `Repository` layer.

At one place we needed to perform a batch task. The task gets generated from a user request.  If the computation is performed on the  request thread it would lead to request timeout.  So we made a decision to use `Async` support provided by Spring. But then we had the following challenge :
>> How do we determine that the operation is done in a separate background service ?

In order to simulate the behaviour i have built the following service :
```
@Component
public class BatchJobService {

    JavaMailSender javaMailSender;
    TaskRepository taskrepo;

    @Async
    public Future<Void> performTask(TaskDetails request) {
        TaskTemplate template = taskRepo.findByTaskName(req.taskName());
        Task task = new  TaskBuilder(template)
                   .withDetails(request)
                   .build();
        TaskResult result = execute(task);
        SimpleMailMessage message = generateMail(result);
        javaMailSender.send(message);
        return new AsyncResult<>(null);
    }

}
```

In the above code we are  creating a task from a template name. The task is executed and then the result is mailed to the user. The whole operation is marked as `Async`.  Testing this code is challenge. If we invoke the code in a unit test we would validate the interactions with the `taskrepo` and the `javaMailSender`. But we can't validate if the operation is performed in background.  

The next approach was to run the solution as a `Spring` test. But then since `Spring` would invoke the method as an `Async` task.  But then the challenge was how to assert method invocation. `Spring` detached method invocation from our  test execution. Thus test turned out to be flaky. In order to make this a repeatable test we modified it with countdown latch. :

```
@SpringBootTest
class BatchJobServiceTest {

    @Test
    void mock_mvc_should_be_set() throws Exception {
        JavaMailSender mock = Mockito.mock(JavaMailSender.class);
        TaskRepository mockrepo = Mockito.mock(TaskRepository.class);
        service.javaMailSender = mock;
        service.taskrepo = mockrepo;
        CountDownLatch latch = new CountDownLatch(1);
        Mockito.doAnswer((x) -> {
            latch.await();
            return null;
        }).when(mock).send(Mockito.any(SimpleMailMessage.class));
        Future<Void> voidFuture = service.perform(new TaskDetails());
        assertThat(voidFuture.isDone()).isFalse();
        latch.countDown();
        assertThat(voidFuture.isDone()).isTrue();
        Mockito.verify(mock).send(Mockito.any(SimpleMailMessage.class))
    }

}
```

In the above test we intercepted the Mockito stub with our implementation. The implementation would `wait` on a `CountDownLatch`. In return we get back a `Future` task which we validate for completion. As a result we can assert with confidence that our Async task is completed within the test. Additionally we added a `Timeout` to the test which would fail the test in finite time  instead of an indefinite wait.