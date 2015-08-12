redux-requests [![Version][npm-image]][npm-url]
===================

Manages in-flight requests with a [Redux](https://github.com/gaearon/redux) [reducer](https://gaearon.github.io/redux/docs/basics/Reducers.html) - avoid issuing duplicate requests without any special logic!

[Live Example](#)

## Avoiding the issue of multiple requests

Say your application has two views for the same set of data, and this data has not yet been fetched. A naïve approach would be to create an [Action Creator](https://gaearon.github.io/redux/docs/basics/Actions.html) which fetches the data from an HTTP API endpoint, and then have both of these views trigger this action as soon as they are rendered (`componentWillMount` in React terms).

The problem with this approach is that **you end up with two identical HTTP requests when you only need one**! You waste bandwidth doing this, and you may also waste render cycles as the [Store](https://gaearon.github.io/redux/docs/basics/Store.html) updates twice as a result of handling both identical responses.

### How can we fix this?

You could wrap all your calls to fetch the data with `if` statements, and keep track of that state somewhere, but who wants to do that by hand?

### Enter redux-requests

This library will not only keep track of all pending requests for you, but also provide a convenient [middleware](https://gaearon.github.io/redux/docs/api/applyMiddleware.html) function that will avoid dispatching Actions to request data if there is already a pending HTTP request in flight!

As a result, you can use the *very same naive approach outlined earlier with hardly any code changes* and it will "just work"! Keep your views stateless and your Reducers ignorant of the notion of "pending requests"!


## Before and after

The only changes you should have to make to your code are:

1. Add `meta.httpRequest` fields to your Action objects
  - `meta.httpRequest.url` is required, and will be used as the unique identifier for the request
  - `meta.httpRequest.done` is a boolean indiecating if this action corresponds to a beginning or ending part of the request sequence
    - Typically a successful response Action, in addition to a failed response Action with an error, will both have `meta.httpRequest.done = true`
2. Check if the `dispatch` for your initial request Action was cancelled (`dispatch` will return `undefined`), and if so do not issue your request
3. Register the `createRequestMiddleware` middleware and the `requestsReducer` reducer as part of your Redux configuration.

#### Original code:

```js
// React component
class Repos extends Component {
  constructor(props) {
    super(props);
  }

  componentWillMount() {
    // Action Creator attempts to request data for this user
    this.props.loadRepos(this.props.username);
  }

  render() {
    return (
      <div>
        { this.props.repos }
      </div>
    );
  }
}

function mapStateToProps(state)  {
  return {
    repos: state.githubRepos
  };
}

function mapDispatchToProps(dispatch) {
  return {
    loadRepos: (userId) => { dispatch(loadRepos(userId)); }
  };
}

export const ReposComponent = connect(mapStateToProps, mapDispatchToProps)(Repos);

// Action Creator
export function loadRepos(userId) {
  return function (dispatch, getState) {
    const url = `https://api.github.com/users/${userId}/repos`;

    dispatch({
      type: 'LOAD_REPOS',
      payload: {
        userId
      }
    });

    fetch(url)
      .then(response => dispatch({
        type: 'LOAD_REPOS',
        payload: {
          userId,
          response
        }
      }))
      .catch(error => dispatch({
        type: 'LOAD_REPOS',
        error: true,
        payload: {
          userId,
          error
        }
      })
    );
  }
}

// Store
const createStoreWithMiddleware = applyMiddleware(thunkMiddleware)(createStore);
let store = createStoreWithMiddleware(combineReducers({ githubRepos }));
```

#### New code:

```js
// React component stays exactly the same!

// Action Creator changes slightly
export function loadRepos(userId) {
  return function (dispatch, getState) {
    const url = `https://api.github.com/users/${userId}/repos`;

    if (!dispatch({
      type: 'LOAD_REPOS',
      payload: {
        userId
      },
      meta: {
        // Add the metadata to the action for the middleware to check
        httpRequest: { url, done: false }
      }
    })) {
      return; // bail out here if the middleware cancelled the dispatch
    }

    const end = { url, done: true };
    fetch(url)
      .then(checkStatus)
      .then(parseJSON)
      .then(addDelay(5000))
      .then(response => dispatch({
        type: 'LOAD_REPOS',
        payload: {
          userId,
          response
        },
        meta: {
          // Add the metadata to the action for the middleware to check
          httpRequest: end
        }
      }))
      .catch(error => dispatch({
        type: 'LOAD_REPOS',
        error: true,
        payload: {
          userId,
          error
        },
        meta: {
          // Add the metadata to the action for the middleware to check
          httpRequest: end
        }
      })
    );
  }
}

// Add new requests store and middleware
import { requestsReducer, createRequestMiddleware } from 'redux-requests';
const createStoreWithMiddleware = applyMiddleware(thunkMiddleware, createRequestMiddleware())(createStore);
let store = createStoreWithMiddleware(combineReducers({ requestsReducer, githubRepos }));
```

## API

### `requestsReducer(state, action)`

A reducer that keeps track of pending request state. It only operates on actions containing the `meta.httpRequest` field.

### `createRequestMiddleware(stateSelectorFunction)`

Returns a middleware function to pass to `applyMiddleware`. Optionally pass a `stateSelectorFunction` which returns where the `requestsReducer` keeps its state in the Store (if not passed, will default to `state => state.requests`).

Ex: `applyMiddleware(createRequestMiddleware(state => state.pendingHttpRequests))(createStore)`


## Credits

- Author: [David Idol](http://daveidol.com)
- License: [MIT](http://opensource.org/licenses/MIT)

Inspired by the [Marty fetch API](http://martyjs.org/guides/fetching-state/index.html).

[npm-image]: https://img.shields.io/npm/v/redux-requests.svg?style=flat-square
[npm-url]: https://www.npmjs.org/package/redux-requests