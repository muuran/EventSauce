---
permalink: /docs/testing/
title: Testing
alternate_title: Testing Aggregates
published_at: 2018-03-03
updated_at: 2018-03-23
---

Event sourced applications are very easy to test. EventSauce ships with test tooling
that allows for scenario based testing using a given/when/then structure. This kind
testing is often associated with Behavior Driven Development (BDD).

Tests written in this style are very expressive and easy to read. This makes it
simple to document business requirements. The test tooling also makes it easy
to follow the TDD approach. Creating tests (and the required events) often gives
very usable insights when modelling your application.

## 1. Create a base test case for your aggregate.

It's advised to create a base test for your aggregate. This base class sets up the defaults
for all test cases around the aggregate. There are a couple of methods that need to be implemented:

* `aggregateRootId` is expected to always return the same aggregate root ID
* `aggregateRootClassName` for the fully qualified aggregate root class name
* `handle` executes the `when` input, usually by passing a command object to a method on the aggregate

You can choose to setup a handle method in your base test case or per test case.
If you use a command based interaction you'll want to set it up in your base class.

```php
use EventSauce\EventSourcing\AggregateRootId;
use EventSauce\EventSourcing\AggregateRootTestCase;

abstract class SignUpProcessTestCase extends AggregateRootTestCase
{
    protected function aggregateRootId(): AggregateRootId
    {
        static $id;
        
        if ($id === null) {
            $id = SignupId::create();
        }
        
        return $id;
    }

    protected function aggregateRootClassName(): string
    {
        return SignUpProcess::class;
    }
    
    protected function handle(/* your arguments for when */)
    {
        // Implement me.
    }
}
```

## 2. Create test scenarios

Now that you've got your base class it's time to set up test scenarios. It's advised to
create small, focused test cases that handle a specific task or condition.

Fir instance, initiating a sign-up process:

```php
class InitiatingSignUpProcessTest extends SignUpProcessTestCase
{
    /**
     * @test
     */
    public function initiating_a_sign_up_process()
    {
        $processId = $this->aggregateRootId();

        $this->when(new InitiateSignUpProcess(
            $processId,
        ))->then(new SignUpProcessWasInitiated());
    } 
}
```

The `when` call takes the input needed to `handle` our scenario. The `then` clause specifies
one or more events that we expect to be recorded afterwards.

In some cases you'll want to record multiple events from a single interaction, this is also
possible:

```php
$this->when(new InitiateSignUpProcess(
    $processId
))->then(
    new SignUpProcessWasInitiated(),
    new AnotherThingHappened()
);
```

If we don't expect something to happen, we can also express that. This allows you to test for
idempotent handling of events:

```php
class InitiatingSignUpProcessTest extends SignUpProcessTestCase
{
    /**
     * @test
     */
    public function not_recording_events()
    {
        $processId = $this->aggregateRootId();

        $this->when(new UneventfulCommand(
            $processId,
        ))->thenNothingShouldHaveHappened();
    }
}
```

The `handle` implementation for this command looks something like:

```php
abstract class SignUpProcessTestCase extends AggregateRootTestCase
{
    protected function handle($command)
    {
        $process = $this->repository->retrieve($command->processId());
        
        if ($command instanceof InitiateSignUpProcess) {
            $process->initiate();           
        }
        
        $this->repository->persist($process);
    }
}
```
