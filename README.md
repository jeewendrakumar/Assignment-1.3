# Assignment-1.3

Architectural overview
Both Redux and Relay were inspired by Flux, an architectural pattern for designing your applications. The basic idea of Flux is to always make your data flow in one direction in the application, from your Stores to your (React) Components. Components call Action Creators, they dispatch Actions that your Stores react to. Flux was originally presented by Facebook, but they didn’t provide a library with it. However, very soon many Flux implementations appeared in the community and many more custom implementations are hidden inside closed-source codebases :). Flux matches the model of React very well, because it tries to prevent data from flowing upward from children to stores, by moving that chain out to action dispatch.

Redux
Redux was born as a very minimal implementation of the Flux pattern. Eschewing the multiple Store pattern from Flux, it has one central store. Store data can be mapped to components via transformation functions. Store can handle actions via reducers. The biggest difference is that a reducer should always be a pure function that takes the old state and an action and returns a new state. In the original Flux, the stores usually mutate their state as a reaction to Actions, in Redux it’s always a functional transformation. Any data can be stored in a Redux store, Redux has no opinion about the data source.

Redux is a very minimal library. Actions can be intercepted and transformed via middlewares and many of them are available from the community, both to make it easier to write your applications in Redux and to provide some complete functionality.

Relay
Relay is in many ways inspired by Flux too. There is a central store, changes to it are made through actions (called Mutations in Relay). However, Relay doesn’t give developers control over the contents of the Store. Rather, Relay leverages GraphQL query language to automatically decompose, store and modify the data received from the server. Components declare their data requirements via GraphQL query fragments and Relay automatically composes the query that fulfills the requirements of all components in the current component tree. Modifications to the Store can be done through a declarative mutation API, but all mutations correspond to server-side mutations. Unlike Redux, you can only store data that has corresponding data on the server in Relay and the server must have a GraphQL API.

Relay provides many features out of the box. Relay handles all the data fetching and makes sure that all (and only) the required data is fetched. Relay has rich support for pagination, especially for use cases isomorphic to infinite scrolling. Relay mutations can perform optimistic updates, report their status and rollback.

Component integration
Redux
Both Relay and Redux provide a very good React integration. Redux is less dependent on React and can be used with other frameworks without a problem. Relay currently can only be used with React (or React Native). However, work is being done on extracting the component layer from Relay, so that it can be used outside React.

Redux encourages separating the presentation and data logic through a concept of smart and dumb components. Usually smart components are created by Redux, they subscribe to a store and can dispatch actions. Dumb components are just normal React components. Smart components describe the data that they require by defining a function that maps store state to their props. Multiple smart components can exists, but usually the component hierarchy should be kept dumb and it should get the data through props.

import { connect } from 'react-redux'

const getVisibleTodos = (todos, filter) => {
  switch (filter) {
    case 'SHOW_ALL':
      return todos
    case 'SHOW_COMPLETED':
      return todos.filter(t => t.completed)
    case 'SHOW_ACTIVE':
      return todos.filter(t => !t.completed)
  }
}

const mapStateToProps = (state) => {
  return {
    todos: getVisibleTodos(state.todos, state.visibilityFilter)
  }
}

const VisibleTodoList = connect(
  mapStateToProps,
)(TodoList)

export default VisibleTodoList
Redux requires a top level component called Provider that will pass all the props needed by Redux smart components into the React context.

render(
  <Provider store={store}>
    <App />
  </Provider>,
  document.getElementById('root')
)
Relay
In Relay most components are smart. Relay wraps React components in Relay Containers. Containers declare what data they need to render with a GraphQL fragment. Containers can compose fragments from their child components, but the actual requirements are opaque. This way containers are isolated from each other and it’s possible to reuse or change them without worrying about data not being available. Resolved fragments are passed to the component as simple props.

class Todo extends React.Component {
  render() {
    return (
      <div>{this.props.todo.id} {this.props.todo.text}</div>
    );
  }
}

const TodoContainer = Relay.createContainer(Todo, {
  fragments: {
    todos: () => Relay.QL`
      fragment on Todo {
        id,
        text
      }
    `,
  }
})
Fragments from the whole component tree are composed together into one query. Data that has already been fetched before will be subtracted from the query and if all the data is available, no query will be made.

const TodoListContainer = Relay.createContainer(Todo, {
  fragments: {
    todoList: () => Relay.QL`
      fragment on TodoConnection {
        count,
        edges {
          node {
            ${TodoContainer.getFragment('todo')}
          }
        }
      }
    `,
  },
})
Relay has a root level component called RootContainer that acts like an entry point. RootContainer requires a Container and a Route. Relay Routes define the start of the query, the actual query root into which the fragments from the whole component tree will be composed into. Route is needed because similar component trees might have different initial source, for example we can imagine a <TodoList /> component that can be rendered both as a list of tasks of the whole team or only as tasks of one person. Routes can also have parameters, for example to be able to pass an object id.

class TodoRoute extends Relay.Route {
  static routeName = 'TodoRoute';

  static queries = {
    todoById: () => Relay.QL`
      query {
        todoById(id: $id)
      }
    `
  };

  static paramDefinitions = {
    id: { required: true }
  };
}

render(
  <Relay.RootContainer
    Component={SingleTodoContainer}
    route={new TodoListRoute} />,
  document.getElementById('root')
)
Mutations
Changing the data in the store is another common task for a client-side data management layer. Often we want to react to user actions fast, to do so-called optimistic updates. After that we will wait for the server to return the result of the actual server call, and if the result is successful, we will confirm our changes.

Let’s make a mutation that changes the text of a TODO item, updates it on the server, and rolls back the change, if it fails.

Redux
We will use the thunk middleware to perform async actions. We will dispatch an optimistic change first, then either roll it back or confirm it.

function todoChange(id, text, isOptimistic) {
  return {
    type: TODO_CHANGE,
    todo: { id, text, isOptimistic }
  };
}

function editTodo(id, text) {
  return (dispatch, getState) => {
    const oldTodo = getState().todos[id];
    // Perform optimistic update
    dispatch(todoChange(id, text, true))
    fetch(`/todo/${id}`, {
      method: 'POST'
    }).then((result) => {
      if (result.code === '200') {
        // Confirm update
        dispatch(todoChange(id, text, false))
      } else {
        dispatch(todoChange(oldTodo.id, oldTodo.text, false))
      }
    })
  }
}

// In the store handler
case TODO_CHANGE:
  return {
    ...state,
    todos: {
      ...state.todos,
      [id]: {
        id,
        text,
        isOptimistic
      }
    }
  }
)

// Now we can dispatch it

store.dispatch(editTodo(todo.id, todo.text));
Relay
There are no custom actions in Relay. Instead, we will need to define a Mutation using Relay mutation DSL. I should note that GraphQL mutations work in an interesting way – mutation is an operation and then a query, so we can request the data with GraphQL mutations in a similar way as with a GraphQL query. This way we can request the data that changed, so that Relay knows what to update.

I’ll go through every element in mutation class in the comments.

class ChangeTodoTextMutation extends Relay.Mutation {
  // Get the name of the mutation on server, so that we can call the server
  getMutation() {
    return Relay.QL`mutation{ updateTodo }`;
  }

  // Map the props passed to mutation to the server input object
  getVariables() {
    return {
      id: this.props.id,
      text: this.props.text,
    };
  }

  // Define a query on the resulting payload, with all the data that changed
  getFatQuery() {
    return Relay.QL`
      fragment on _TodoPayload {
        changedTodo {
          id
          text
        }
      }
    `;
  }

  // Define what exactly Relay should change in the store. In this case
  // we say that it should match the item from `changedTodo` element in the
  // result with an item in the store by id and then update the item in the
  // store
  getConfigs() {
    return [{
      type: 'FIELDS_CHANGE',
      fieldIDs: {
        changedTodo: this.props.id,
      },
    }];
  }

  // To make Relay make an optimistic update, we need to "fake" the response
  // from the server. Here it's pretty easy.
  getOptimisticResponse() {
    return {
      changedTodo: {
        id: this.props.id,
        text: this.props.text,
      },
    };
  }
}
Now this mutation can be dispatched in a similar way as an action.

Relay.Store.commitUpdate(
  new ChangeTodoTextMutation({
    id: this.props.todo.id,
    text: text,
  }),
);
We can also pass callbacks to commitUpdate to react to the failure or success of the mutation. However, in any case, the rollback will happen automatically, if the request fails.

Relay.Store.commitUpdate(
  new ChangeTodoTextMutation({
    id: this.props.todo.id,
    text: text,
  }), {
   onFailure: () => {
     console.error('error!');
   },
   onSuccess: (response) => {
     console.log('success!')
   }
});
Mutation DSL is considered by some a weak spot of Relay. Sometimes it’s not easy to figure out all the parameters and which exact query Relay expects. It should be noted that Relay can do some pretty complicated things, like doing mutations on items that are members of some paginated list data. However, Relay maintainers are currently working on a new lower level API that will, hopefully, improve the experience of writing mutations.

Working with paginated data
Redux
There is no one way to do pagination in Redux – just as much as there is no one way to do it in RESTful APIs. To implement pagination in Redux you need to create a reducer that keeps track of what has been fetched so far, what’s the cursor (or a URL or page number) of the next page to fetch and if more data is currently being fetched.

There is an example implementation of pagination with Redux in the real-world example app, which paginates responses from the GitHub API.

Redux is at disadvantage here, because the lack of standardized server API means the framework can not handle pagination automatically. However, this also means that by manually implementing pagination, you can work with APIs with various conventions and store the data in Redux.

Relay
Relay compatible GraphQL APIs have a concept of a Connection, an abstraction over listed data. The key part of that is that all connections in Relay GraphQL accept pagination parameters and all items of a connection have a cursor, which serves as a pointer to paginate the connection in relation to the item. In addition, connections have a pageInfo field that contains information whether there is a next page available.

The benefit of having a standardized API for pagination is that Relay will do most of the hard work for us. We simply update the parameters, and it will fetch the items we don’t have yet.

To use a cursor, we’ll use variables of the Relay Containers, which can be changed from within the Container. Variables can be passed to the GraphQL fragments.

Let’s consider a simple PaginatedTodoList component and a Container.

class PaginatedTodoList extends React.Component {
  nextPage() {
    const lastElement = this.props.user.todos.edges.length - 1;
    this.setVariables({
      after: this.props.user.todos.edges[lastElement].cursor;
    });
  }

  render() {
    return (
      <div>
        <TodoList todos={this.props.user.todos} />
        {this.props.user.todos.pageInfo.hasNextPage ?
        <button onClick={this.nextPage} :
        <div>Last page</div>}
      </div>
    );
  }
}

const PaginatedTodoListContainer = Relay.createContainer(PaginatedTodoList, {
  fragments: {
    user: () => Relay.QL`
      fragment on User {
        todos(first: 10, after: $after) {
          edges {
            cursor
          }
          ${TodoList.getFragment('todos')}
          pageInfo {
            hasNextPage
          }
        }
      }
    `,
  },
  initialVariables: {
    after: null,
  },
});
When the user clicks the button to load more items, we just update the after variable, Relay will fetch any missing data and the list will be updated.
