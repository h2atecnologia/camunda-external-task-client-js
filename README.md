# camunda-external-task-client

A practical JavaScript client for [Camunda External Tasks](https://docs.camunda.org/manual/latest/user-guide/process-engine/external-tasks/).

> NodeJS >= v8.9.4 is required

## About External Tasks
External Tasks are service tasks whose execution differs particularly from the execution of other service tasks (e.g. Human Tasks).
The execution works in a way that a list of workers polls units of work from the engine and complete them.

**camunda-external-task-client.js** allows you to create easily such workers in NodeJS. 

## Features
* [Fetch and Lock](https://docs.camunda.org/manual/latest/reference/rest/external-task/fetch/) 
* [Complete](https://docs.camunda.org/manual/latest/reference/rest/external-task/post-complete/)
* [Handle Failure](https://docs.camunda.org/manual/latest/reference/rest/external-task/post-failure/) 
* [Handle BPMN Error](https://docs.camunda.org/manual/latest/reference/rest/external-task/post-bpmn-error/)
* [Unlock](https://docs.camunda.org/manual/latest/reference/rest/external-task/post-unlock/)
* [Extend Lock](https://docs.camunda.org/manual/latest/reference/rest/external-task/post-extend-lock/) 


## Installing

```sh
npm install -s camunda-external-task-worker-js
```

Or:

```sh
yarn add camunda-external-task-worker-js
```

## Usage

1. Create a simple process model with an External Service Task and define the topic as 'topicName'.
2. Deploy the process to the Camunda BPM engine.
3. In your NodeJS script:

```js
const { Workers, logger } = require('camunda-external-task-worker-js');

// Create a Workers instance and configure it to use a logger and a path to the Camunda engine Rest API
const workers = new Workers({ use: logger, path: 'http://localhost:8080/engine-rest' });

// Susbscribe a worker to the topic: 'topicName'
workers.subscribe('topicName', ({ task, taskClient }) => {
  // Put your business logic
    
  // Complete the task
  taskClient.complete(task);  
});
```
  

## API

* [Workers](#workers)
* [new Workers([options])](#new-workersoptions)
  * [workers.start()](#workersstart)
  * [workers.subscribe(topic, [options], worker)](#workerssubscribetopic-options-worker)
  * [workers.stop()](#workersstop)
* [BasicAuthInterceptor](#basicauthinterceptor)
  * [new BasicAuthInterceptor([options])](#new-basicauthinterceptoroptions)
* [logger](#logger)

### Workers
```js
const { Workers } = require('camunda-external-task-worker-js');
```

Workers is the core class of the external task client. 
It is used to start/stop the client and subscribe a worker to a certain topic.

#### new Workers([options])
Options are **mandatory** when creating a _Workers_ instance.
Here's a list of the available options:

|        Option        |                                                                             Description                                                                            | Type                   | Required |      Default     |
|:--------------------:|:------------------------------------------------------------------------------------------------------------------------------------------------------------------:|------------------------|:--------:|:----------------:|
| path                 | Path to the engine api                                                                                                                                             | string                 | ✓        |                  |
| workerId             | The id of the worker on which behalf tasks are fetched. The returned tasks are locked for that worker and can only be completed when providing the same worker id. | string                 |          | 'some-random-id' |
| maxTasks             | The maximum number of tasks to fetch                                                                                                                               | number                 |          | 10               |
| interval             | Interval of time to wait before making a new poll.                                                                                                                 | number                 |          | 300              |
| lockDuration         | The default duration to lock the external tasks for in milliseconds.                                                                                               | number                 |          | 50000            |
| autoPoll             | If true, then polling start automatically as soon as a Workers instance is created.                                                                                | boolean                |          | true             |
| asyncResponseTimeout | The Long Polling timeout in milliseconds.                                                                                                                          | number                 |          |                  |
| interceptors         | Function(s) that will be called before a request is sent. Interceptors receive the configuration of the request and return a new configuration.                    | function or [function] |          |                  |
| use                  | Function(s) that have access to the workers instance as soon as it is created and before any polling happens.                                                      | function or [function] |          |                  |

#####  _About interceptors_
- Interceptors receive the configuration of the request and return a new configuration. 
- In the case of multiple interceptors, they are piped in the order they are provided. 
- Check out [BasicAuthInterceptor](/lib/BasicAuthInterceptor.js) for a better understanding of the usage of interceptors.   

##### _About use_
Check out [logger](/lib/logger.js) for a better understanding of the usage of middlewares. 

#### workers.start()
Triggers polling. 

##### _About Polling_
- Polling tasks from the engine works by performing a fetch & lock operation of tasks that have registered workers
  to them. It then calls the worker registered to each task.
- Polling is done periodically based on the _interval_ configuration.  
- [Long Polling](https://docs.camunda.org/manual/latest/user-guide/process-engine/external-tasks/#long-polling-to-fetch-and-lock-external-tasks) is enabled by configuring the option _asyncResponseTimeout_.
 
#### workers.subscribe(topic, [options], worker)
Subscribes a worker to a specific topic and returns a _worker client_.
Here's a list of the available parameters:

| Parameter     | Description                                           | Type     | Required | Default |
|---------------|-------------------------------------------------------|----------|----------|---------|
| topic         | topic name for which external tasks should be fetched | string   | ✓        |         |
| options       | options about subscription                            | object   |          |         |
| worker        | worker function                                       | function | ✓        |         |

The only possible options supported now are:

| Option       | Description                                           | Type   | Required | Default                                                |
|--------------|-------------------------------------------------------|--------|----------|--------------------------------------------------------|
| lockDuration | specifies the lock duration for this specific worker. | number |          | global lockDuration configured in the workers instance |

##### _About the worker function_

```js
const worker = ({ task, taskClient }) => {
  // doing some work
  
  // 1- worker can complete a task:
  taskClient.complete(task);
  
  // 2- worker can handleFailure of a task:
  taskClient.handleFailure(task, 'some failure message');
  
  // 3- worker can handleBPMNFailure of a task:
  taskClient.handleBPMNFailure(task, 'some BPMN failure message');
  
  // 4- worker can extendLock of a task:
  taskClient.extendLock(task, 5000);
};

workers.subscribe('bar', worker);
```

The worker function receives an object that has the following parameters:
- _task_: task object locked by the worker. For more information about the task object, check out this section of [Camunda Docs](https://docs.camunda.org/manual/develop/reference/rest/external-task/fetch/).
- _taskClient_: object that provides methods to perform the following operations on a specific task.
    

##### _About worker client_
A worker client, which is returned by the **subscribe()** method, is a an object that provides the following:
- **worker:** the worker function.
- **unsubscribe():** a method to unsubscribe the worker.
- **lockDuration:** the lockDuration for the worker.

```js
const { Workers } = require('camunda-external-task-worker-js');

const workers = new Workers({ path: 'http://localhost:8080/engine-rest' });

const workerClient = workers.subscribe('foo', ({ task, taskClient }) => {
  // do some foo work
});

// unsubscribes the worker
workerClient.unsubscribe();
```

#### workers.stop()
Stops polling.

### BasicAuthInterceptor
A BasicAuthInterceptor instance is a simple interceptor that adds basic authentication to all requests.

```js
const { Workers, BasicAuthInterceptor } = require('camunda-external-task-worker-js');

const basicAuthentication = new BasicAuthInterceptor({ username: 'demo', password: 'demo' });

const workers = new Workers({ path: 'http://localhost:8080/engine-rest',  interceptors: basicAuthentication });
```


#### new BasicAuthInterceptor([options])
Here's a list of the available options:

| Option   | Description                           | Type   | Required | Default |
|----------|---------------------------------------|--------|----------|---------|
| username | username used in basic authentication | string | ✓        |         |
| password | password used in basic authentication | string | ✓        |         |

### logger
A logger is a simple middleware that logs various events in the workers lifecycle.

```js
const { Workers, logger } = require('camunda-external-task-worker-js');

const workers = new Workers({ use: logger, path: 'http://localhost:8080/engine-rest' });
```

## License
Unless otherwise specified this project is licensed under [Apache License Version 2.0](./LICENSE).
