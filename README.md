# Customhooks

Custom hooks are reusable function that contain state logic.

Unlike regular function, custom hooks can use other React Hooks and React state.


# Create custom Hooks - counter

Before : 

```javascript 
import { useState, useEffect } from 'react';
 
import Card from './Card';
import useCounter from '../hooks/use-counter'
 
const ForwardCounter = () => {
 
  useCounter()
 
  const [counter, setCounter] = useState(0);
 
  useEffect(() => {
    const interval = setInterval(() => {
      setCounter((prevCounter) => prevCounter + 1);
    }, 1000);
 
    return () => clearInterval(interval);
  }, []);
 
  return <Card>{counter}</Card>;
};
 
export default ForwardCounter;
```


The name of the custom hooks must start with ‘use’.

Then return the state.

```javascript
import {useState, useEffect} from 'react'
 
const useCounter = () => {
    const [counter, setCounter] = useState(0);
 
    useEffect(() => {
      const interval = setInterval(() => {
        setCounter((prevCounter) => prevCounter + 1);
      }, 1000);
 
      return () => clearInterval(interval);
    }, []);
 
    return counter
}
 
export default useCounter;
```

Then we use the custom hooks, and we can remove the state and the useEffect, so it becomes :

```javascript
import Card from './Card';
import useCounter from '../hooks/use-counter'
 
const ForwardCounter = () => {
 
  const counter = useCounter()
 
  return <Card>{counter}</Card>;
};
 
export default ForwardCounter;
```

The custom hook is tied to the component where it is used (incl. The state and effect in the custom hook), it is not shared accord components.

# Create custom Hooks - Http request

## Before : 

GET Request

```javascript
import React, { useEffect, useState } from 'react';
 
import Tasks from './components/Tasks/Tasks';
import NewTask from './components/NewTask/NewTask';
 
function App() {
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState(null);
  const [tasks, setTasks] = useState([]);
 
  const fetchTasks = async (taskText) => {
    setIsLoading(true);
    setError(null);
    try {
      const response = await fetch(
        'https://react-http-26861-default-rtdb.firebaseio.com/tasks.json'
      );
 
      if (!response.ok) {
        throw new Error('Request failed!');
      }
 
      const data = await response.json();
 
      const loadedTasks = [];
 
      for (const taskKey in data) {
        loadedTasks.push({ id: taskKey, text: data[taskKey].text });
      }
 
      setTasks(loadedTasks);
    } catch (err) {
      setError(err.message || 'Something went wrong!');
    }
    setIsLoading(false);
  };
 
  useEffect(() => {
    fetchTasks();
  }, []);
 
  const taskAddHandler = (task) => {
    setTasks((prevTasks) => prevTasks.concat(task));
  };
 
  return (
    <React.Fragment>
      <NewTask onAddTask={taskAddHandler} />
      <Tasks
        items={tasks}
        loading={isLoading}
        error={error}
        onFetch={fetchTasks}
      />
    </React.Fragment>
  );
}
 
export default App;
 
```

POST request

```javascript
import { useState } from 'react';
 
import Section from '../UI/Section';
import TaskForm from './TaskForm';
 
const NewTask = (props) => {
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState(null);
 
  const enterTaskHandler = async (taskText) => {
    setIsLoading(true);
    setError(null);
    try {
      const response = await fetch(
        'https://react-http-26861-default-rtdb.firebaseio.com/tasks.json',
        {
          method: 'POST',
          body: JSON.stringify({ text: taskText }),
          headers: {
            'Content-Type': 'application/json',
          },
        }
      );
 
      if (!response.ok) {
        throw new Error('Request failed!');
      }
 
      const data = await response.json();
 
      const generatedId = data.name; // firebase-specific => "name" contains generated id
      const createdTask = { id: generatedId, text: taskText };
 
      props.onAddTask(createdTask);
    } catch (err) {
      setError(err.message || 'Something went wrong!');
    }
    setIsLoading(false);
  };
 
  return (
    <Section>
      <TaskForm onEnterTask={enterTaskHandler} loading={isLoading} />
      {error && <p>{error}</p>}
    </Section>
  );
};
 
export default NewTask;
 
```

## After 

Custom Hook

```javascript
import {useState, useCallback} from 'react'

const useHttp = () => {
    const [isLoading, setIsLoading] = useState(false);
    const [error, setError] = useState(null);
  
    const sendRequest = useCallback(async (requestConfig, applyData) => {
      setIsLoading(true);
      setError(null);
      try {
        const response = await fetch(
            requestConfig.url, {
                method: requestConfig.method ? requestConfig.method : 'GET',
                headers: requestConfig.headers ? requestConfig.headers : {},
                body: requestConfig.body ? JSON.stringify(requestConfig.body) : null
            }
        );
  
        if (!response.ok) {
          throw new Error('Request failed!');
        }
  
        const data = await response.json();
        applyData(data)
       
      } catch (err) {
        setError(err.message || 'Something went wrong!');
      }
      setIsLoading(false);
    },[]);

    return {
        isLoading: isLoading,
        error: error,
        sendRequest: sendRequest,
    }
}

export default useHttp;
```

GET request

```javascript
import React, { useEffect, useState } from 'react';

import Tasks from './components/Tasks/Tasks';
import NewTask from './components/NewTask/NewTask';
import useHttp from './hooks/use-http';

function App() {
  const [tasks, setTasks] = useState([]);



  const {isLoading, error, sendRequest: fetchTasks} = useHttp();

  useEffect(() => {
    const transformTasks = (tasksObj) => {
      const loadedTasks = [];
  
        for (const taskKey in tasksObj) {
          loadedTasks.push({ id: taskKey, text: tasksObj[taskKey].text });
        }
  
        setTasks(loadedTasks);
    }

    fetchTasks({url: 'https://react-http-26861-default-rtdb.firebaseio.com/tasks.json'}
    ,transformTasks);
  }, [fetchTasks]);

  const taskAddHandler = (task) => {
    setTasks((prevTasks) => prevTasks.concat(task));
  };

  return (
    <React.Fragment>
      <NewTask onAddTask={taskAddHandler} />
      <Tasks
        items={tasks}
        loading={isLoading}
        error={error}
        onFetch={fetchTasks}
      />
    </React.Fragment>
  );
}

export default App;

```

POST request

```javascript
import Section from '../UI/Section';
import TaskForm from './TaskForm';
import useHttp from '../../hooks/use-http';

const NewTask = (props) => {
  const {isLoading, error, sendRequest: sendTaskRequest} = useHttp();
 
  const createTask = (taskText, taskData) => {
    const generatedId = taskData.name; // firebase-specific => "name" contains generated id
      const createdTask = { id: generatedId, text: taskText };

      props.onAddTask(createdTask);
  }

  const enterTaskHandler = async (taskText) => {
    sendTaskRequest({
      url:  'https://react-http-26861-default-rtdb.firebaseio.com/tasks.json',
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: { text: taskText }
    }, createTask.bind(null,taskText) );

  };

  return (
    <Section>
      <TaskForm onEnterTask={enterTaskHandler} loading={isLoading} />
      {error && <p>{error}</p>}
    </Section>
  );
};

export default NewTask;

```