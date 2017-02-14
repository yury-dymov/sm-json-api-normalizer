# json-api-normalizer: Easy Way To Integrate JSON API And Redux
![JSON API + redux](https://habrastorage.org/files/325/507/e71/325507e71b1c43399d89018f3601ae6f.jpg)

[JSON API](http://jsonapi.org/) standard is becoming more and more popular nowadays. From my perspective, it is a very convenient solution for representing data, which helps to focus on business logic rather than on developing parsers and serializers on server and client sides over and over again.

# JSON API vs. Typical Web-Services
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
      "text": "Great job, Bro!"
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


# Editor Note: This is where you should put a bit more about why you're using Redux. The *Note directly below doesn't quite offer the detail I feel is needed to best articulate that.

*Note: I am integrating JSON API with redux in this article. However, you may use your favorite state management library instead. According to the recent survey [State of JavaScript in 2016](http://stateofjs.com/2016/statemanagement/), redux is almost holding a monopoly in this field. Therefore I decided to stick with it.*

# Redux Best Practices
## 1. Keep Data Flat In The Redux Store
Many JSON documents have a deeply nested structure. If you prefer to store your data in a similar manner in the store, you will sooner or later face many issues. For example, you might store the same object several times. For example, you might have a Post and Comment objects, which share the same Author. Your store will look like this:

```JSON
{
  text: "My Post",
  author: {
    name: "Yury",
    avatar: "avatar1.png"
  },
  comments: [
    {
      text: "Awesome Comment",
      author: {
   		 name: "Yury",
	    avatar: "avatar1.png"      
      }
    }
  ]
}
```

As you can see, we store the same Author Object in several places, which not only requires more memory but also has negative side-effects. Imagine, in the backend somebody changed the user's avatar. Instead of updating one object in the redux store, you now need to traverse the whole state and update all instances of the same object. Not only it might be very slow, but also requires you to learn precisely the data object structure. Refactoring will be a nightmare as well. Another issue is that if you decide to reuse certain data objects for new views, and they would be nested into some other objects, traversal implementation would also be complex, slow and dirty.

Instead, we can store the data in a flatten structure. This way each object will be stored only once, and we will have a very easy access to any data.

```JSON
{
  post: [{
    id: 1,
    text: "My Post",
    author: { id: 1 },
    comments: [ { id: 1 } ]
  }],
  comment: [{
    id: 1,
    text: "Awesome Comment"
  }],
  author: [{
    name: "Yury",
    avatar: "avatar1.png",
    id: 1
  }]
 }
```

Same principles are widely used in [RDMBS](https://en.wikipedia.org/wiki/Relational_database_management_system) for many years.

## 2. Store Collections As Maps Whenever Possible

# Editor Note: I think this sections needs to be fleshed out more and offer more details about the differences in performance and use cases. This feels like a passing comment & doesn't quite explain to the reader "why?".

If data order in the collection is not important, then Map data structure will work a lot better compared to Array from the performance perspective.

| Type | Add | Delete | Search |
------|:---------------:|:-----------:| :-----------: |
| Map | O(1) | O(1) | O(1) |
| Array | O(1)| O(n) | O(n) |

It is worth to mention that the merge operation will also work a lot faster for Maps as it can be broken down to many Search + Add operations.


# JSON API And Redux

# Editor Note: Reading through I'm not convinced of the following statement because I feel there's still more that needs to be outlined as to why you're choosing Redux as well as why the best practices matter within the context of this article and demo app.

After we learned, how data is structured with JSON API and redux best practices, it should be clear that there is a perfect match between two out-of-the-box. However, there are still some things left, which we can improve. 

Despite the fact that dividing data objects into two types `data` and `included` might make some sense for the application, we can't afford to store them as two separate entities in the redux store as otherwise the same data objects will be stored more than one time, which violates redux best practiсes.

As we discussed, JSON API also returns a collection of objects in the form of an array but for the redux store using maps is a lot more suitable option.

To fill this gap, I have recently developed [json-api-normalizer](https://github.com/yury-dymov/json-api-normalizer) library.

Main features:

1. Merging `data` and `included` fields, normalizing the data
2. Collections are converted into maps in a form `id` => `object`
3. Original response structure is stored in a special `meta` object
4. One-to-many relationships are merged into one object with serialized IDs

Let me explain 3d and 4th points.

First of all, distinguishing `data` and `included` data objects was introduced in JSON API specification to resolve issues with recursive structures and circular dependencies. Second, most of the times, data in redux is incrementally updated, which helps to improve performance and implement offline support. However, as we work with the same data objects in our application, sometimes it is not possible to distinguish, which data objects should we use for a particular view. 
json-api-normalizer can store web service response structure in a special `meta` field, so you can unambiguously determine, which data objects were fetched for a specific API request.

json-api-normalizer is also converting one-to-many relationships from JSON API format

```JSON
{
  "relationships": {
    "comments": [{
      "type": "comment",
      "id": "1",
    }, {
      "type": "comment",
      "id": "2",    
    }, {
      "type": "comment",
      "id": "3",    
    }]
  }
}
```

to this format

```JSON
{
  "relationships": {
    "comments": {
      "type": "comment",
      "id": "1,2,3"
    }
  }
}
```

From my experience, this format is more suitable for redux as you can easily update the data and relationships with one `merge`, where you need to specify a new string of IDs, rather than explicitly finding in array required an object and deleting it manually. However, this form might not suit every use-case, so if you feel that you want to override my format, feel free to send your pull request.

# Practical Example

# Editor's Note: I think this section should be clearer giving more context as to what's being built and what to expect. For example, I feel that the reference to the API is not clear enough and while you don't need to go in-depth on Pheonix since this is front-end specific, I think you should be clear that you're using it and what the results from the API will be. Basically, don't leave it to the reader to guess.

*Note: I assume that you have some practical experience with React and Redux so far.*

We are going to build a very simple Web Application. Imagine, we have a backend, which provides us several types of data objects with several kinds of relationships between them. 

![relationships](https://github.com/yury-dymov/phoenix-json-api-example/raw/master/docs/diagram.png)

Here we have a Question data object, which might have many Post objects. Each Post might have many Comment objects. Each Post and Comment has one Author respectfully. 

Our Web App will perform a request to the backend, store fetched data in the redux store and render the content on the page.

You can review the web service output [here](https://phoenix-json-api-example.herokuapp.com/api/test), which will be converted to something like this.

# Editor's Note: Missing image here
![live](https://api.monosnap.com/rpc/file/download?id=bz5z1RhUNU4vonY5OGDuU52cFuhXQp)

Live demo is also available [here](https://yury-dymov.github.io/json-api-react-redux-example/).

So, we need a backend with a JSON API support. As this article is fully dedicated to the frontend development, I prebuilt publically available data source so that we can focus on our Web App. If you are interested, you may check [the source code](https://github.com/yury-dymov/phoenix-json-api-example). Note that there are many [JSON API implementation libraries](http://jsonapi.org/implementations/) available for all kind of technology stacks, so feel free to choose one, which will work best for you.

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

Question visualization component:
#### src/components/Question/package.json
```JSON
{
  "name": "Question",
  "version": "0.0.0",
  "private": true,
  "main": "./Question"
}
```

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

Post visualization component:
#### src/components/Post/package.json
```JSON
{
  "name": "Post",
  "version": "0.0.0",
  "private": true,
  "main": "./Post"
}
```

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

User visualization component:
#### src/components/User/package.json
```JSON
{
  "name": "User",
  "version": "0.0.0",
  "private": true,
  "main": "./User"
}
```

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

Comment visualization component:
#### src/components/Comment/package.json
```JSON
{
  "name": "Comment",
  "version": "0.0.0",
  "private": true,
  "main": "./Comment"
}
```

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

# Editor's Note: This sentence seems very abrupt with no real closure. I think a better call to action or guidance is merited here. 

And we are done! If something doesn't work for you, feel free to compare your code with [my project master branch](https://github.com/yury-dymov/json-api-react-redux-example)

Live demo is also available [here](https://yury-dymov.github.io/json-api-react-redux-example)

# Conclusion
json-api-normalizer и redux-object libraries were developed very recently. They might look very simple, but it took me lots of time and experience to eventually come up with such implementation. By using them, you can avoid lots of traps and pitfalls I faced in the last year. So I am quite sure that these simple tools might be very helpful for the community and they will help to save a lot of time for many people.

Feel free to join the discussion and help me to improve these tools.


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
