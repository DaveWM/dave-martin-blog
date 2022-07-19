+++
date = 2022-07-14T23:00:00Z
description = "A critique of React's hooks API, and an explanation of why it's insufficient as a web development framework"
draft = true
title = "Critique of Pure Hooks"

+++
As you may have guessed from the title, I'm not a big fan of React's hooks API. I'm not _absolutely_ against hooks, and I do think they can be used effectively as part of a larger framework. However it's my strongly held opinion that in and of themselves, hooks are not sufficient to build complex web apps.

I've now worked on 3 React applications which heavily used hooks. Each time the code quickly became bloated, difficult to understand, and full of bugs. I'm very much against the, seemingly ubiquitous, belief that hooks obviate any need to architect your application. Nothing could be further from the truth. In this post, I hope to convince you of that.

### Why Hooks?

Firstly, allow me to steelman the argument for hooks. Hooks were designed to be more functional, composable, and terser than React's class API. To some extent, it has been successful in this. Say you have a `Todo` component written using the class API:

```javascript
export default class Todo extends React.Component {
  constructor(props) {
      super(props);
      this.state = {
          text: null
      };
  }

  render() {
      return (
        <div>
          <input type="text" onChange={(e) => this.setState({text: e.target.value})} />
          <span>Value is: {this.state.text}</span>
        </div>
      );
  }
}
```

Using hooks, you can shorten it to this:

```javascript
function Todo(){
  const [text, setText] = useState(null);
  return (
    <div>
      <input type="text" onChange={(e) => setText(e.target.value)} />
      <span>Value is: {text}</span>
    </div>
  );
}
```

This is fewer lines of code, and also clearer. You don't need any knowledge about classes, constructors, inheritance, or the `super` keyword. Hooks allow you to write components as a function. You don't have to think about lifecycle methods (`componentDidMount`, `componentDidUpdate`, etc.). Hooks are more composable and reduce repetition.

### Difficulties with Hooks

Unfortunately when you overuse hooks, things quickly go awry. Taking our example from above, let's add an input property and a simple side effect:

```javascript 
function Todo({done}){
  const [text, setText] = useState(null);
  useEffect(() => {
    console.log(`Todo ${text}, ${done ? "done" : "in progress"}`);
  }, [text]);
  
  return (
    <div>
      <input type="text" onChange={(e) => setText(e.target.value)} />
      <span>Value is: {text}</span>
    </div>
  );
}
```

At first glance this looks great. However, on closer inspection we've introduced a bug! We're reading the `done` property in the `useEffect` function, but we haven't added it to the dependencies array. This means the effect won't be run when `done` changes. This may seem trivial to fix, but it can be extremely difficult to track down bugs like this. In my experience the dependencies array is an endless source of bugs. Linters can help, [but don't eliminate](https://typeofweb.com/wady-react-hooks#eslint) these bugs. Another "gotcha" occurs if you use anything other than a primitive value as a dependency. This is never mentioned in the documentation. I have personally spent many hours tracking down these types of bugs, and this seems like a common experience among all React developers.

Unfortunately, that's far from the only drawback of hooks. Testing is always tricky when you have local state in components, but hooks add to this difficulty. With the class API, it was often possible to set a component's state then assert that the output of `render` is correct. In general, with hooks this isn't possible. This is because you can use multiple `useState` hooks within a single component. You are forced to cajole the component into the right state, often by simulating a sequence of UI actions. This is time consuming, tedious, error prone, and leads to bloated tests. Also, you are often forced to add delays, which drastically slow down your tests.

Another major hurdle is the infamous "Rules of Hooks". These approximate to "the number and order of hooks for a given component has to remain constant". In practice, this means that you can't conditionally use a hook. You have to put the conditional in a function, that is passed to the hook. This isn't natural, and can make your code difficult to understand. If you need a variable number of hooks, you have to wrap each one in a child component. This can lead to a proliferation of mostly useless child components, and can disrupt your carefully designed component hierarchy. The root cause of these restrictions is that React tracks hooks by the order in which they are called. I understand the reasoning behind this design choice, but it leads to a lot of problems nonetheless.

While it's true that hooks are more composable than classes, they are less so than functions. Consider the following component, that makes 2 effects - `foo` and `bar`:

```js
function useDoSomething(message) {
  useEffect(() => {
   setTimeout(() => {
     console.log(message);
   }, Math.random() * 1000) 
  }, []);
}

function App() {
  useDoSomething("foo"); // foo
  useDoSomething("bar"); // bar

  return <div>App</div>;
}
```

There is no output of the `foo` and `bar` effects. We just want to trigger them when the component is mounted. Now suppose we want to trigger a third effect, `baz`, once `foo` and `bar` have completed. This is surprisingly difficult to do using hooks. We have to add an output to `useDoSomething`, and then use it to conditionally execute `baz` like so:

```js
function useDoSomething(message) {
  const [done, setDone] = useState(false);
  useEffect(() => {
   setTimeout(() => {
     console.log(message);
     setDone(true);
   }, Math.random() * 1000) 
  }, []);
  return done;
}

function useDoSomethingElse(message, dependencies) {
  useEffect(() => {
    if(dependencies.every(x => x)){
      console.log(message);
    }
  }, dependencies);
}

function App() {
  const foo = useDoSomething("foo");
  const bar = useDoSomething("bar");

  useDoSomethingElse("baz", [foo, bar]);

  return <div>App</div>;
}
```

This works, but it's counterintuitive to say the least. Similar code using promises or observables is far more understandable. The reason for this, fundamentally, is that composing hooks is difficult. The only tools you have for composition are state and the dependencies array. In some situations, they can be used to craft elegant solutions. However, in the vast majority of situations this way of writing code is unnatural and cumbersome.

The drawbacks I've outlined above are major hinderances, but ones that can in principle be surmounted. If using hooks naturally led to a good architecture, it may be worth putting up with these problems. Unfortunately, I don't believe this is the case. To explain why, I'll first have to explain a bit about state management.

### State Management

Managing state is one of the central problems in programming. Every application must do it in some way. Over the years, every approach imaginable to UI state management has been tried. Two broad categories of architectures emerged: object oriented (OO), and functional.

OO architectures include MVC/MVVM frameworks like Backbone, Ember, and Angular. It is characterised by storing state locally in components, and allowing it to be freely mutated. OO frameworks provide tools to manage this local state, and synchronise it between components. These include dependency injection, 2-way binding, services/factories, and event buses. Although I do think OOP has its flaws, when done well it does allow you to write relatively clear, testable code.

Functional architectures usually look something like the flux architecture (although there are variations, such as [Purescript's Halogen](https://purescript-halogen.github.io/purescript-halogen/index.html)). They are characterised by keeping state in as few places outside of components, in as few places as possible. Mutations of this state are tightly controlled. Components are ideally pure functions, which requires extracting side effects to another place in the code. Using this style of architecture makes it easy to write, debug, and test your application. Since pure functions are used as much as possible, your code becomes highly composable with as loose coupling.

## The main problem with hooks

This brings me to the main reason I'm against hooks. When you rely entirely on hooks you inevitably end up with an OO architecture, but without any of the tools you need to manage and synchronise local state. Hooks push you towards storing mutable state in components. This state often needs to be synchronised with the other components, but React doesn't provide any tools to help with this. You're left with just props and callbacks. Components can freely make side effects, which makes testing very difficult. You need dependency injection to make testing feasible, but React doesn't provide it. You also need a way of setting a component's internal state, but again React won't help you here.

This is a far cry from what React was originally intended for. React was never intended to be used as a framework. It was designed as the view layer of a functional architecture - [the "V" in MVC](https://web.archive.org/web/20140321012426/http://facebook.github.io/react). Facebook used (and still uses, as far as I know) React as the view layer of their flux architecture. Lee Byron, a developer on the React team, explained this at length in a 2013 [Quora post](https://www.quora.com/How-is-Facebooks-React-JavaScript-library-How-does-it-compare-with-other-popular-JavaScript-libraries/answer/Lee-Byron). One of the React team's earliest blog posts, published in June 2013, states:

> React is a library for building composable user interfaces. It encourages the creation of reusable UI components which present data that changes over time.

React was initially introduced as an alternative to increasingly overbearing UI frameworks like AngularJS. These frameworks had become large, complex, and cumbersome. Unfortunately, now it seems that the prevailing belief is that React should be used as a framework. I don't think anything could be further from the truth.

### Comparison with AngularJS

To demonstrate my point, let's compare some modern React code to good old AngularJS:

```js
function Counter() {
  const [counter, setCounter] = useState(0);

  useEffect(() => {
    console.log(counter);
  }, [counter]);

  return (<div>
    <span>Count is: {counter}</span>
    <button onClick={() => setCounter(counter + 1)}>Clicky</button>
  </div>);
}
```

```js
// Template
<body ng-app="app" ng-controller="CounterController">
  <span>Count is: {{counter}}</span>
  <button ng-click="counter = counter + 1">Clicky</button>
</body>

// Controller
angular.controller("CounterController", function($scope){
  $scope.counter = 0;
  $scope.watch('counter', (val) => console.log(val));
});
```

The overall approach of each snippet is similar. The component holds state (`counter`), which is mutated when the button is clicked. This `counter` state is bound to the rendered HTML. The component reacts when its state is updated, and makes a side effect.

As well as the similarity, it's interesting to note that the AngularJS is shorter and more straightforward. Going beyond this toy example, AngularJS provided many tools and conveniences that React does not. AngularJS gave you dependency injection, 2-way binding (including with child components), services/factories, and even an event bus in the form of `$emit`. These all helped you constrain the complexity inherent in using local state and side effects. It's also interesting to note that using multiple chained `$scope.$watch`s was always deemed an antipattern. By contrast, hooks both encourage and require this pattern.

To be clear, I'm not advocating that Google resurrect AngularJS. My point is that React has come full circle. It now very much resembles the frameworks it sought to make redundant, except without the conveniences they afforded.

## A Better Way

I've been painting a pretty bleak picture so far. However, the good news is there are solutions. The key is to reduce your reliance on hooks. One option is to use a framework. There are many to choose from, and it's not too difficult to add them to an existing codebase. You can't go far wrong with Redux and Redux Toolkit. If you prefer the OOP approach, there are many great MVC/MVVM frameworks out there. If you'd like to try a different language, take a look at Elm or ClojureScript with Re-frame.

Before I close, I'd just like to point out that it hasn't been my intention to disparage the React devs. As I mentioned above, I think hooks are a good innovation in many ways, and a definite improvement on the class API. I hope I've made it clear that what I'm against is the _overuse_ of hooks, and the belief that they will solve all your problems. Thanks for reading!