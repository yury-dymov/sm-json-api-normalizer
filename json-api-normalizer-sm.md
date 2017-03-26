# json-api-normalizer: Easy Way To Integrate JSON API And Redux
![JSON API + redux](https://habrastorage.org/files/325/507/e71/325507e71b1c43399d89018f3601ae6f.jpg)

As a Front-end Developer, for each and every application I work on I need to decide, how to manage the data. The problem can be broken down to the three following subproblems:

1) Fetching data from the backend
2) Storing it somewhere locally in the front-end application
3) Retrieving the data from the local store and format it in a way required for the particular view or screen

This article sums up my experiences with consuming data from JSON, JSON API, and GraphQL backends, and gives practical recommendations on how to manage the front-end application data.

To illustrate my ideas and make an article closer to the real-world use-cases, I would develop a very simple frontend application by the end of the article. The scenario is following: imagine, we implement a survey, which asks the same pile of questions many users. After each user provides his or her answers, other users might comment them if desired. Our Web App will perform a request to the backend, store fetched data in the local store and render the content on the page. To keep things simple, we will omit the answer creation flow this time.

![live](https://habrastorage.org/files/d6e/c8b/321/d6ec8b321dbc481ab628b266e6de08da.png)

Live demo is also available [here](https://yury-dymov.github.io/json-api-react-redux-example/).

# Backstory

Within the last couple of years, I participated in many frontend projects based on React stack. We are using Redux for the state management not only because it is the most widely-used solution in this category according to the recent [State of JavaScript in 2016](http://stateofjs.com/2016/statemanagement/) survey, but it is also very lightweight, straightforward and predictable. Yes, sometimes it is required to write a lot more boilerplate code compared to other available state management solutions but on the other hand, you can fully understand and control how your application works, which gives you a lot of freedom to implement any business logic and scenarios. 

To give you the context, some time ago we tried [GraphQL](http://graphql.org/) and [Relay](https://facebook.github.io/relay/) in one of our Proof-of-Concept projects. Don't get me wrong, it works great, but every time we wanted to implement slightly different flow from the standard one, we were fighting with our stack instead of delivering new features. I know that many things changed since then, and Relay is a decent solution now, but we learned the hard way that using simple and predictable tools work better for us as we can plan our development process more precisely and meet our deadlines. 

*Note: before moving forward I assume that you have some basic knowledge on state management and either Flux or Redux*

# Redux Best Practices

The best things about Redux is that it is unopinionated on what kind of API you consume. You can even change your API from JSON to JSON API or GraphQL and back during the development and in case your preserve your data model, this will not affect your state management implementation at all. This is possible because before you put API response to the store you should process it in a certain way. Redux itself doesn't force you to do that but community discovered and developed several best practises based on real-world experience. Following them will help you to save a lot of time by reducing complexity of your applications and decrease number of bugs and corner cases.

## 1. Keep Data Flat In The Redux Store
Let's go back to the demo application scenario and discuss the Data Model:

![relationships](https://github.com/yury-dymov/phoenix-json-api-example/raw/master/docs/diagram.png)

Here we have a Question data object, which might have many Post objects. Each Post might have many Comment objects. Each Post and Comment has one Author respectfully. 

Let's assume; we have a backend, which returns a typical JSON response. Very likely it would have a deeply nested structure. If you prefer to store your data in a similar manner in the store, you will sooner or later face many issues. For example, you might store the same object several times. For example, you might have a Post and Comment objects, which share the same Author. Your store will look like this:

```JSON
{
  "text": "My Post",
  "author": {
    "name": "Yury",
    "avatar": "avatar1.png"
  },
  "comments": [
    {
      "text": "Awesome Comment",
      "author": {
            "name": "Yury",
        "avatar": "avatar1.png"      
      }
    }
  ]
}
```

As you can see, we store the same Author Object in several places, which not only requires more memory but also has negative side-effects. Imagine, in the backend somebody changed the user's avatar. Instead of updating one object in the redux store, you now need to traverse the whole state and update all instances of the same object. Not only it might be very slow, but also requires you to learn precisely the data object structure. Refactoring will be a nightmare as well. Another issue is that if you decide to reuse certain data objects for new views, and they would be nested into some other objects, traversal implementation would also be complex, slow and dirty.

Instead, we can store the data in a flatten structure. This way each object will be stored only once, and we will have a very easy access to any data.

```JSON
{
  "post": [{
    "id": 1,
    "text": "My Post",
    "author": { "id": 1 },
    "comments": [ { "id": 1 } ]
  }],
  "comment": [{
    "id": 1,
    "text": "Awesome Comment"
  }],
  "author": [{
    "name": "Yury",
    "avatar": "avatar1.png",
    "id": 1
  }]
 }
```

Same principles are widely used in [RDMBS](https://en.wikipedia.org/wiki/Relational_database_management_system) for many years.

## 2. Store Collections As Maps Whenever Possible

Ok, so we got the data in a nice flat structure. It is a very common practice to incrementally accumulate received data in either Redux store and/or local storage so that we can re-use it later as a cache to improve the performance or for the offline usage.

However, after merging new data to the existing storage, we need to select only relevant data objects for the particular view, not everything we received so far. To achieve that, we can store the structure of each JSON document separately, so we can quickly figure out, which data objects were provided within a particular request. This structure will contain a list of the data object IDs, which we can use to fetch the data from the storage.

Let me illustrate this point. We will perform two requests to fetch a list of friends of two different users Alice and Bob and review the content of our storage accordingly. To make things easier, let's assume that in the beginning, the storage is empty.

#### /alice/friends Response
```JSON
{
  "data": [{
    "type": "User",
    "id": "1",
    "attributes": {
      "name": "Alice"
    }
  }]
}
```

So, here we are getting User data object with ID "1" and name "Mike," which might be stored like this:

#### Storage State #1
```JSON
{
  "users": [
    {
      "id": "1",
      "name": "Mike"
    }
  ]
}
```

Another request will return us User with ID "2" and name "Kevin":

#### /bob/friends Response
```JSON
{
  "data": [{
    "type": "User",
    "id": "2",
    "attributes": {
      "name": "Kevin"
    }
  }]
}
```

After merge our storage will look like this:

#### Storage State #2
```JSON
{
  "users": [
    {
      "id": "1",
      "name": "Mike"
    },
    {
        "id": "2",
        "name": "Kevin"
    }
  ]
}
```

And the big question is, how can we distinguish from this point, which users are Alices' friends and which are Bobs'?

For example, we could preserve the structure of the JSON API document, so we might quickly figure out, which data objects in the storage are relevant. Keeping this in mind, we can change the implementation of the storage so that it might look like this:

#### Storage State #2 With Metadata
```JSON
{
  "users": [
    {
      "id": "1",
      "name": "Mike"
    },
    {
        "id": "2",
        "name": "Kevin"
    }
  ],
  "meta": {
      "/alice/friends": [
        {
          "type": "User",
          "id": "1"
        }          
      ],
      "/bob/friends": [
        {
          "type": "User",
          "id": "2"
        }
      ]
  }
}
``` 

Now, we can read the metadata and fetch all mentioned data objects. Problem solved! Can we do better? Note that we constantly doing three operations: insert, read and merge. Which data structure will work the best for us from the performance perspective?

Let's briefly recap operation complexities.

| Type | Add | Delete | Search | Preserves Order |
------|:---------------:|:-----------:| :-----------: | :-----------: |
| Map | O(1) | O(1) | O(1) | No |
| Array | O(1)| O(n) | O(n) | Yes |

*Note: if you are note very familiar with Big O Notation, "n" here means the number of data objects, O(1) means that operation will take relatively the same amount of time regardless of the data set size, and O(n) means that operation execution time is linearly dependent on the data set size. You can read more on Big O Notation [here](https://en.wikipedia.org/wiki/Big_O_notation)* 

As we can see, Maps will work a lot better than Arrays as all operations have complexity O(1) instead of O(n). If data object order is important, we still can use Maps for the data handling and save the ordering information in the metadata.

Let me reimplement storage mentioned above and use Map over Array for the User data object.

#### Storage State #2 Revised
```JSON
{
  "users": {
      "1": {
        "name": "Mike"
      },
      "2": {
        "name": "Kevin"
      }
  },
  "meta": {
      "/alice/friends": [
        {
          "type": "User",
          "id": "1"
        }          
      ],
      "/bob/friends": [
        {
          "type": "User",
           "id": "2"
        }
      ]
  }
}
``` 

Now, instead of iterating over the whole array to find the certain user, you can get it by ID almost instantly.

# Processing The Data And JSON API
As you can imagine, there should be a widely-used solution to convert JSON documents to the Redux-friendly form. [Normalizr](https://github.com/paularmstrong/normalizr) library was initially developed by Dan Abramov, the author of Redux, for this particular purpose. You have to provide JSON document and the scheme to `normalize` function, and it will return the data in a nice flat structure, which we can save in the Redux store. 

We used this approach in many projects, and while it works great if you know your data model in advance, which doesn't change much within application lifecycle, it dramatically fails if things are too dynamic. For example, when you are prototyping, developing a Proof-of-Concept or a new product, the data model changes very frequently to fulfill new requirements and change requests. Each backend change should be reflected in a normalizr scheme update. Because of this, I ended up fighting with my frontend app to fix things rather than working on new features several times.

Are there any alternatives? We tried out GraphQL and JSON API. 

While GraphQL seems very promising and might be an interesting choice, we were not able to adopt it at that time as our APIs are consumed by many 3d Parties, and we can't just drop the REST approach. 

Let's briefly discuss [JSON API](http://jsonapi.org/) standard.

## JSON API vs. Typical Web-Services
JSON API main features:

* data is represented in a flat structure with no more, than one level deep relationships
* data objects are typified
* JSON API specification defines pagination, sorting and data filtering features out-of-the-box

#### Typical JSON Document
```JSON
{
  "id": "123",
  "author": {
    "id": "1",
    "name": "Paul"
  },
  "title": "My awesome blog post",
  "comments": [
    {
      "id": "324",
      "text": "Great job, Bro!",
      "commenter": {
        "id": "2",
        "name": "Nicole"
      }
    }
  ]
}
```

#### JSON API Document
```JSON
{
  "data": [{
     "type": "post",
     "id": "123",
     "attributes": {
         "id": 123,
         "title": "My awesome blog post"
     },
     "relationships": {
         "author": {
           "type": "user",
           "id": "1"
         },
         "comments": {
           "type":  "comment",
           "id": "324"
         }
     }     
  }],
  "included": [{
      "type": "user",
      "id": "1",
      "attributes": {
        "id": 1,
        "name": "Paul"
      }
  }, {
    "type": "user",
    "id": "2",
    "attributes": {
      "id": 2,
      "name": "Nicole"
    }
  }, {
    "type": "comment",
    "id": "324",
    "attributes": {
      "id": 324,
      "text": "Great job!"
    },
    "relationships": {
      "commenter": {
        "type": "user",
        "id": "2"
      }
    }
  }]
 }
```

It might seem that JSON API is too verbose compared to the traditional JSON, isn't it?

| Type | Raw (bytes) | Gzipped (bytes) |
------|:---------------:|:-----------:|
| Typical JSON | 264 | 170 |
| JSON API| 771 | 293 |

The size difference in a raw format is very remarkable, but gzipped versions are much closer to each other.

Keep in mind that it is also possible to develop a synthetic example, which will have a greater size in a Typical JSON format compared to the JSON API. Imagine dozens of blog posts, which share the same author. In a Typical JSON document, you will have to store Author object for each Post Object and in the JSON API format Author object will be stored only once.

Bottomline, yes, the size of JSON API document on average is bigger, but it shouldn't be considered as an issue. Typically, you are dealing with structured data, which compresses five or more times, and also relatively small, thanks to pagination.

Let's discuss the advantages.

* First of all, JSON API returns data in a flat form with no more than one level of relationships. This helps to avoid redundancy and guarantees that each unique object will be stored in a document only once. This approach is a perfect match for redux best practices, and we will utilize this feature soon.
* Second, data is provided in the form of typified objects, which means that on the client side you don't need to implement parsers or define schemes like you do with [normalizr](https://github.com/paularmstrong/normalizr). This makes your front-end apps more prone to the data structure changes and requires fewer efforts from your side to adapt your application for the new requirements.
* Third, JSON API specification defines "links" object, which helps to move pagination, filtering and sorting feature implementation from your application to the JSON API clients. There is also an optional "meta" object available, where you can define your app-specific payload.



## JSON API And Redux

How two work together? Absolutely great!

JSON API provides data in a flat structure by definition, which conforms nicely with Redux best practices. Data comes typified so that it can be naturally saved in the Redux storage in a Map with the format "type" => Map of objects. Are we missing something?

Despite the fact that dividing data objects into two types `data` and `included` might make some sense for the application, we can't afford to store them as two separate entities in the redux store as otherwise the same data objects will be stored more than one time, which violates redux best practiсes.

As we discussed, JSON API also returns a collection of objects in the form of an array but for the redux store using maps is a lot more suitable option.

To fill this gap, I have recently developed [json-api-normalizer](https://github.com/yury-dymov/json-api-normalizer) library.

Main features:

1. Merging `data` and `included` fields, normalizing the data
2. Collections are converted into maps in a form `id` => `object`
3. Original response structure is stored in a special `meta` object

First of all, distinguishing `data` and `included` data objects was introduced in JSON API specification to resolve issues with recursive structures and circular dependencies. Second, most of the times, data in redux is incrementally updated, which helps to improve performance and implement offline support. However, as we work with the same data objects in our application, sometimes it is not possible to distinguish, which data objects should we use for a particular view. 
json-api-normalizer can store web service response structure in a special `meta` field, so you can unambiguously determine, which data objects were fetched for a specific API request.

# Demo App Implementation

*Note: I assume that you have some practical experience with React and Redux so far.*

Once again, we will build a very simple web app, which will render the survey data provided by backend in a JSON API format. 

First of all, we need a backend with a JSON API support. As this article is fully dedicated to the frontend development, I prebuilt publically available data source so that we can focus on our Web App. If you are interested, you may check [the source code](https://github.com/yury-dymov/phoenix-json-api-example). Note that there are many [JSON API implementation libraries](http://jsonapi.org/implementations/) available for all kind of technology stacks, so feel free to choose one, which will work best for you.

My demo web service provides us two questions. First one has two answers, second — three. The second answer to the first question has three comments.

You can review the web service output [here](https://phoenix-json-api-example.herokuapp.com/api/test), which will be converted to something similar to this after the user presses the button, and data is successfully fetched.

## 1. Download The Boilerplate
To reduce time on Web App configuration, I developed [small React boilerplate](https://github.com/yury-dymov/json-api-react-redux-example/tree/initial), which can be used as a starting point.

Let's clone the repo.

```Bash
git clone https://github.com/yury-dymov/json-api-react-redux-example.git --branch initial
```

Now we have:

* React and ReactDOM
* Redux and Redux DevTools
* Webpack
* Eslint
* Babel
* Entry point to the application, two simple components, eslint config, webpack config and redux store initialization
* I also defined CSS for all components, which we are going to develop

Everything should work out-of-the-box with no actions needed from your side.

To start the application type in the console

```Bash
npm run webpack-dev-server
```

and open in the browser [`http://localhost:8050`](http://localhost:8050).

## 2. API Integration
Let's start with developing a redux middleware, which will interact with API. We will use json-api-normalizer here to stick with Don't Repeat Yourself principle, as otherwise, we will have to use it over and over again in many redux actions.

#### src/redux/middleware/api.js
```JavaScript
import fetch from 'isomorphic-fetch';
import normalize from 'json-api-normalizer';

const API_ROOT = 'https://phoenix-json-api-example.herokuapp.com/api';

export const API_DATA_REQUEST = 'API_DATA_REQUEST';
export const API_DATA_SUCCESS = 'API_DATA_SUCCESS';
export const API_DATA_FAILURE = 'API_DATA_FAILURE';

function callApi(endpoint, options = {}) {
  const fullUrl = (endpoint.indexOf(API_ROOT) === -1) ? API_ROOT + endpoint : endpoint;

  return fetch(fullUrl, options)
    .then(response => response.json()
      .then((json) => {
        if (!response.ok) {
          return Promise.reject(json);
        }

        return Object.assign({}, normalize(json, { endpoint }));
      }),
    );
}


export const CALL_API = Symbol('Call API');

export default function (store) {
  return function nxt(next) {
    return function call(action) {
      const callAPI = action[CALL_API];

      if (typeof callAPI === 'undefined') {
        return next(action);
      }

      let { endpoint } = callAPI;
      const { options } = callAPI;

      if (typeof endpoint === 'function') {
        endpoint = endpoint(store.getState());
      }

      if (typeof endpoint !== 'string') {
        throw new Error('Specify a string endpoint URL.');
      }

      const actionWith = (data) => {
        const finalAction = Object.assign({}, action, data);
        delete finalAction[CALL_API];
        return finalAction;
      };

      next(actionWith({ type: API_DATA_REQUEST, endpoint }));

      return callApi(endpoint, options || {})
        .then(
          response => next(actionWith({ response, type: API_DATA_SUCCESS, endpoint })),
          error => next(actionWith({ type: API_DATA_FAILURE, error: error.message || 'Something bad happened' })),
        );
    };
  };
}
```

After data is returned from API and parsed we convert it to the redux-friendly format with json-api-normalizer and forward it further to the redux actions.

*Note: this code was copy-pasted from [redux real-world example](https://github.com/reactjs/redux/blob/master/examples/real-world/src/middleware/api.js) with small adjustments to add json-api-normalizer. Now you can see that integration with json-api-normalizer is very simple and straightforward.*

Let's adjust redux store configuration:

#### src/redux/configureStore.js
```JavaScript
...
+++ import api from './middleware/api';

export default function (initialState = {}) {
  const store = createStore(rootReducer, initialState, compose(
--- applyMiddleware(thunk),  
+++ applyMiddleware(thunk, api),
    DevTools.instrument(),
...    
```

Now we can implement our first action, which will request data from the backend:

#### src/redux/actions/post.js
```JavaScript
import { CALL_API } from '../middleware/api';

export function test() {
  return {
    [CALL_API]: {
      endpoint: '/test',
    },
  };
}
```

Let's implement the reducer, which will merge provided data from the backend into the redux store:

#### src/redux/reducers/data.js
```JavaScript
import merge from 'lodash/merge';
import { API_DATA_REQUEST, API_DATA_SUCCESS } from '../middleware/api';

const initialState = {
  meta: {},
};

export default function (state = initialState, action) {
  switch (action.type) {
    case API_DATA_SUCCESS:
      return merge(
        {},
        state,
        merge({}, action.response, { meta: { [action.endpoint]: { loading: false } } }),
      );
    case API_DATA_REQUEST:
      return merge({}, state, { meta: { [action.endpoint]: { loading: true } } });
    default:
      return state;
  }
}
```
Now we need to add our reducer to the root reducer:

#### src/redux/reducers/data.js
```JavaScript
import { combineReducers } from 'redux';
import data from './data';

export default combineReducers({
  data,
});
```

Model layer is done! Let's add the button, which will trigger `fetchData` action and download some data for our App.

#### src/components/Content.jsx
```JavaScript
import React, { PropTypes } from 'react';
import { connect } from 'react-redux';
import Button from 'react-bootstrap-button-loader';
import { test } from '../../redux/actions/test';

const propTypes = {
  dispatch: PropTypes.func.isRequired,
  loading: PropTypes.bool,
};

function Content({ loading = false, dispatch }) {
  function fetchData() {
    dispatch(test());
  }

  return (
    <div>
      <Button loading={loading} onClick={() => { fetchData(); }}>Fetch Data from API</Button>
    </div>
  );
}

Content.propTypes = propTypes;

function mapStateToProps() {
  return {};
}

export default connect(mapStateToProps)(Content);
```

Let's open our page in the browser — with the help of Browser DevTools and Redux DevTools we can see that application fetches the data from the backend in a JSON API document format, converts it to a more suitable representation and stores it in the redux store. Great! Everything works as expected, so let's add some UI components to visualize the data.

## 3. Fetching The Data From The Store
[redux-object](https://github.com/yury-dymov/redux-object) package converts the data from the redux store into JSON object. We need to pass part of the store, object type, and ID and it will take care of the rest.

```JavaScript
import build, { fetchFromMeta } from 'redux-object';

console.log(build(state.data, 'post', '1')); // ---> Post Object: { text: "I am fine", id: 1, author: @AuthorObject }
console.log(fetchFromMeta(state.data, '/posts')); // ---> array of posts
```

All relationships are represented as JavaScript Object Properties with lazy loading support. So all child objects will be loaded only when required.

```JavaScript
const post = build(state.data, 'post', '1'); // ---> post object; `author` and `comments` properties are not loaded yet

post.author; // ---> User Object: { name: "Alice", id: 1 }
```


Let's add several UI components to visualize the data.

Typically, React component structure follows the data model and our App is not an exception. 

![React Component Structure](https://habrastorage.org/files/734/6c8/7b1/7346c87b155140c48b48613957dfefad.png)

First, we need to fetch the data from the store and propogate it to the component via `connect` function from `react-redux`:

#### src/components/Content.jsx
```JavaScript
import React, { PropTypes } from 'react';
import { connect } from 'react-redux';
import Button from 'react-bootstrap-button-loader';
import build from 'redux-object';
import { test } from '../../redux/actions/test';
import Question from '../Question';

const propTypes = {
  dispatch: PropTypes.func.isRequired,
  questions: PropTypes.array.isRequired,
  loading: PropTypes.bool,
};

function Content({ loading = false, dispatch, questions }) {
  function fetchData() {
    dispatch(test());
  }

  const qWidgets = questions.map(q => <Question key={q.id} question={q} />);

  return (
    <div>
      <Button loading={loading} onClick={() => { fetchData(); }}>Fetch Data from API</Button>
      {qWidgets}
    </div>
  );
}

Content.propTypes = propTypes;

function mapStateToProps(state) {
  if (state.data.meta['/test']) {
    const questions = (state.data.meta['/test'].data || []).map(object => build(state.data, 'question', object.id));
    const loading = state.data.meta['/test'].loading;

    return { questions, loading };
  }

  return { questions: [] };
}

export default connect(mapStateToProps)(Content);
```

We are fetching object IDs from metadata of the API request with '/test' endpoint, building JavaScript Objects with `redux-object` library and providing them to our component in the "questions" prop.

# Editor's Note: This is where I mentioned in my email that I'd like to see more details about what you're building. Following you have several components with only a one-liner to describe what they are. It'd be great for the reader to have more details. 

# Yury's Note: I added some information, but surprisingly it is not that easy. As components are simple, it is hard to write more than one sentence.

Now we need to implement a bunch of "dumb" components for rendering questions, posts, comments and users. They are very straightforward.

Question visualization component package.json:
#### src/components/Question/package.json
```JSON
{
  "name": "question",
  "version": "0.0.0",
  "private": true,
  "main": "./Question"
}
```

Question Component renders the question text and the list of answers.
#### src/components/Question/Question.jsx
```JavaScript
import React, { PropTypes } from 'react';
import Post from '../Post';

const propTypes = {
  question: PropTypes.object.isRequired,
};

function Question({ question }) {
  const postWidgets = question.posts.map(post => <Post key={post.id} post={post} />);

  return (
    <div className="question">
      {question.text}
      {postWidgets}
    </div>
  );
}

Question.propTypes = propTypes;

export default Question;
```

Post Component package.json:
#### src/components/Post/package.json
```JSON
{
  "name": "post",
  "version": "0.0.0",
  "private": true,
  "main": "./Post"
}
```

Post Component renders some information about the author, answer text and also the list of comments;
#### src/components/Post/Post.jsx
```JavaScript
import React, { PropTypes } from 'react';
import Comment from '../Comment';
import User from '../User';

const propTypes = {
  post: PropTypes.object.isRequired,
};

function Post({ post }) {
  const commentWidgets = post.comments.map(c => <Comment key={c.id} comment={c} />);

  return (
    <div className="post">
      <User user={post.author} />
      {post.text}
      {commentWidgets}
    </div>
  );
}

Post.propTypes = propTypes;

export default Post;
```

User Component package.json:
#### src/components/User/package.json
```JSON
{
  "name": "user",
  "version": "0.0.0",
  "private": true,
  "main": "./User"
}
```

User Component renders some meaningful information about either answer or comment author. In this app, we will output just user's name, but in the real world, we could add an avatar and other nice things for a better User Experience.
#### src/components/User/User.jsx
```JavaScript
import React, { PropTypes } from 'react';

const propTypes = {
  user: PropTypes.object.isRequired,
};

function User({ user }) {
  return <span className="user">{user.name}: </span>;
}

User.propTypes = propTypes;

export default User;
```

Comment Component package.json:
#### src/components/Comment/package.json
```JSON
{
  "name": "comment",
  "version": "0.0.0",
  "private": true,
  "main": "./Comment"
}
```

Comment Component is very similar to the Post Component. It will render some information about the author and the comment's text.
#### src/components/Comment/Comment.jsx
```JavaScript
import React, { PropTypes } from 'react';
import User from '../User';

const propTypes = {
  comment: PropTypes.object.isRequired,
};

function Comment({ comment }) {
  return (
    <div className="comment">
      <User user={comment.author} />
      {comment.text}
    </div>
  );
}

Comment.propTypes = propTypes;

export default Comment;
```

And we are done! Open the browser, press the button, and enjoy the result. 

If something doesn't work for you, feel free to compare your code with [my project master branch](https://github.com/yury-dymov/json-api-react-redux-example)

Live demo is also available [here](https://yury-dymov.github.io/json-api-react-redux-example)

# Conclusion
So this ends up the story I would like to tell. This approach help us to prototype a lot faster and be very prone to data model changes. As data comes out typified and in a flat structure from the backend, we don't need to know in advance relationships between data objects and exact fields. Data will be saved in the Redux store in a format, which conforms to the Redux best practices anyway. This helps us to dedicate most of our time on the feature development and experiments rather than adopting normalizr schemes, rethinking selectors, and debugging over and over again.

I encourage you to try JSON API in your next pet project and ensure that you can spend more time on experiments without fear of breaking things :)

# Links
1. [JSON API Specs](http://jsonapi.org/)
2. [JSON API Implementations](http://jsonapi.org/implementations/)
3. [json-api-normalizer repo](https://github.com/yury-dymov/json-api-normalizer)
4. [redux-object repo](https://github.com/yury-dymov/redux-object)
5. [JSON API Data Source example developed with Phoenix Framework](https://phoenix-json-api-example.herokuapp.com/api/test)
6. [JSON API Data Source example source code](https://github.com/yury-dymov/phoenix-json-api-example)
7. [React Application consuming JSON API Live Demo](https://yury-dymov.github.io/json-api-react-redux-example)
8. [React Application source code, initial version](https://github.com/yury-dymov/json-api-react-redux-example/tree/initial)
9. [React Application source code, final version](https://github.com/yury-dymov/json-api-react-redux-example)