+++
date = 2022-07-25T23:00:00Z
description = "A critique of modern React, specifically the hooks API, and an explanation of why it's insufficient as a web development framework"
title = "Critique of Pure Hooks"

+++
It may surprise you, given the title of this post, that I'm not _absolutely_ against hooks. I actually think that they're mostly an improvement over the old class API. What concerns me is how they've affected how React is used in practice. React is now perceived as a full framework, rather than just a library for UI rendering. It's claimed that hooks have enabled this shift. I've been told that hooks solve fundamental problems like state management, controlling side effects, and writing testable code. I'm dubious of these claims. I've now encountered this belief several times, at several different companies, and it has inspired me to write this post as a rebuttal.

### Why Hooks?

Firstly, allow me to steelman the argument for hooks. Hooks were designed to be more functional, composable, and terser than React's class API. I would say it has been successful in this regard. Say you have a `Todo` component written using the class API:

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

Unfortunately, when you overuse hooks things quickly go awry. Taking our example from above, let's add an input property and a simple side effect:

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

At first glance this looks great. However, on closer inspection we've introduced a bug! We're reading the `done` property in the `useEffect` function, but we haven't added it to the dependencies array. This means the effect won't be run when `done` changes. This may seem trivial to fix, but it can be extremely difficult to track down bugs like this. In my experience the dependencies array is an endless source of bugs. Linters can help, [but don't eliminate](https://typeofweb.com/wady-react-hooks#eslint) these bugs. Another "gotcha" occurs if you use anything other than a primitive value as a dependency. I have personally spent many hours tracking down these types of bugs, and this seems like a common experience among all React developers.

Unfortunately, that's far from the only drawback of hooks. Testing is always tricky when you have local state in components, but hooks add to this difficulty. With the class API, it was often possible to set a component's state then assert that the output of `render` is correct. In general, with hooks this isn't possible. This is because you can use multiple `useState` hooks within a single component. You are forced to cajole the component into the right state, often by simulating a sequence of UI actions. This is time consuming, tedious, error prone, and leads to bloated tests. Also, you are often forced to add delays, which drastically slow down your tests.

Another major hurdle is the infamous ["Rules of Hooks"](https://reactjs.org/docs/hooks-rules.html). These approximate to "the number and order of hooks for a given component has to remain constant". In practice, this means that you can't conditionally use a hook. You have to put the conditional in a function, that is passed to the hook. This isn't natural, and can make your code difficult to understand. If you need a variable number of hooks, you have to wrap each one in a child component. This can lead to a proliferation of mostly useless child components, and can disrupt your carefully designed component hierarchy. The root cause of these restrictions is that React tracks hooks by the order in which they are called. I understand the [reasoning behind this design choice](https://overreacted.io/why-do-hooks-rely-on-call-order/), but it leads to a lot of problems nonetheless.

While it's true that hooks are more composable than classes, they are less so than regular functions. Consider the following component, that performs 2 effects - `foo` and `bar`:

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

This works, but is counterintuitive to say the least. Similar code using promises or observables is far easier to write, and more understandable. The only tools you have for composing hooks are `useState` and the dependencies array. In some situations, these can be used to craft elegant solutions. However, often it forces you to write code in a style that is unnatural and verbose.

The drawbacks I've outlined above are major hinderances, but ones that can in principle be surmounted. It would be worth putting up with these difficulties if hooks naturally lead to a great architecture, or gave you some other big advantage. However, I don't believe this is the case. To explain why, I'll first have to explain a bit about state management.

### State Management

Managing state is one of the fundamental problems in programming. Every application must do it in some way. Over the years, every approach imaginable to UI state management has been tried. Two broad categories of architectures emerged: object oriented (OO), and functional.

OO architectures include MVC and MVVM frameworks like Vue and Angular. It is characterised by storing state locally in components, and allowing it to be freely mutated. OO frameworks provide tools to manage this local state, and synchronise it between components. These include dependency injection, 2-way binding, built in services/factories, and event buses. Although I do think OOP has its flaws, when done well it does allow you to write relatively clear, testable code.

Functional architectures usually look something like the [Flux architecture](https://reactjs.org/blog/2014/05/06/flux.html), as exemplified by [Redux](https://redux.js.org/). They are characterised by keeping state outside of components, in as few places as possible. Mutations of this state are tightly controlled, usually using [CQRS](https://www.eventstore.com/cqrs-pattern). Components are ideally pure functions, which requires extracting side effects to another place in the code. Using this style of architecture makes it easy to write, debug, and test your application. Since pure functions are used as much as possible, it allows you to write highly composable code with loose coupling.

### Not-so-modern React

So that brings us to a question - where does the "hooks architecture" land in this spectrum? I think the best way to answer this is with a comparison. For a basic `Counter` component, let's compare modern React to another framework. The grandfather of JS frameworks. A framework that did OO back when it was cool (well, slightly less uncool). Yes that's right - AngularJS.

_React_

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

_AngularJS_

    // Controller
    angular.controller("CounterController", function($scope){
      $scope.counter = 0;
      $scope.$watch('counter', (val) => console.log(val));
    });
    
    // Template
    <div ng-controller="CounterController">
      <span>Count is: {{counter}}</span>
      <button ng-click="counter = counter + 1">Clicky</button>
    </div>

It's readily apparent how similar the 2 snippets are. Aside from the template being separate in AngularJS, the snippets are almost the same line-for-line. In each, the component holds state (`counter`), which is mutated when the button is clicked. This `counter` state is bound to the rendered HTML. The component reacts when its state is updated, and logs out the current count. This similarity would seem to suggest that modern React is more towards the OO end of the state management spectrum.

As well as the similarity, it's interesting to note that the AngularJS code is shorter and more straightforward. The `ng-click` handler has a more natural and JS-like syntax than the `onClick` handler in React. It's also easier to test, because you can more easily mock the injected `$scope` than the global `useState`.

Going beyond this toy example, AngularJS provided many tools and conveniences that React does not. AngularJS gave you dependency injection, which is completely missing from React. AngularJS also gave you 2-way binding (including with child components), a way of managing services and factories, and even an event bus in the form of `$emit`. These all helped you constrain the complexity inherent in using local state and side effects.

To be clear I'm not advocating that Google resurrect AngularJS, or that it's better than React. I'm just pointing out how much "modern" React resembles it. However, it seems to be missing several critical features that AngularJS provided. Why are these features missing? How have we gone _backwards_, in many ways, in the 10+ years since AngularJS was first released?

## The Big Problem

This brings me to the main reason I'm against hooks. When you rely entirely on hooks you inevitably end up with a half-baked OO architecture. Hooks push you towards this, but React doesn't give you any of the tools or conveniences this approach requires. 

Hooks encourage storing mutable state in components, which often needs to be synchronised with other components. However, the only tool React provides you for this is callbacks. Components can freely make side effects using `useEffect`. However this makes testing nigh-on impossible, because React doesn't give you dependency injection. Hooks allow you to chain state updates and effects to your heart's content, but React doesn't provide a way to make this manageable or test this logic independently.

How did React get in this state? I believe the root cause is that it was never intended to be used as a full framework, but is now being treated as one. React was originally supposed to solely be used for DOM rendering - [the "V" in MVC](https://web.archive.org/web/20140321012426/http://facebook.github.io/react). React was initially introduced as an alternative to increasingly large, complex, and cumbersome frameworks. The common refrain was ["React is a view library, not a framework"](https://web.archive.org/web/20140516230615/http://facebook.github.io:80/react/). Lee Byron, a developer on the React team, explained this at length in a 2013 [Quora post](https://www.quora.com/How-is-Facebooks-React-JavaScript-library-How-does-it-compare-with-other-popular-JavaScript-libraries/answer/Lee-Byron). One of the React team's [earliest blog posts](https://reactjs.org/blog/2013/06/05/why-react.html#react-isnt-an-mvc-framework), published in June 2013, states (emphasis mine):

> React is a library for building composable user interfaces. It encourages the creation of reusable UI components which **present data** that changes over time.

In other words, React _just_ renders data to the page in an efficient way - nothing more. This is what React was intended for, and where it really shines. It should be used as one part, the view layer, of a larger application. Component-local state should only be used as a last resort, for behaviour that doesn't need testing. Hooks don't change any of this. However, it's a distressingly common belief that hooks have turned React into a framework. This is simply incorrect.

## A Better Way

I've been painting a pretty bleak picture so far. However, the good news is there are solutions. The most important thing is to think about how to architect your application.

If you prefer a functional architecture, you can immediately take some steps towards it. Try to pull state and side effects up the component hierarchy as high as possible. You can also need to strictly control how child components can update this state. The best way to do this is by having them dispatch events to the state-holding parent component, which determines how to update the state. React gives you the [useReducer](https://reactjs.org/docs/hooks-reference.html#usereducer) hook to help with this. Eventually, you'll be able to separate out state and side effects entirely from your React code.

It may be easier to use a framework, in which case you can't go far wrong with [Redux](https://redux.js.org/) and [Redux Observable](https://redux-observable.js.org/). Redux is fairly easy to introduce gradually to an existing codebase. It will instantly reduce your reliance on hooks and local mutable state, and make your app more testable and maintainable.

Unfortunately it's a bit more difficult to update an existing React app to use a more OOP approach. However, for greenfield projects there are many great MVC/MVVM frameworks to choose from. [Angular](https://angular.io/) and [Vue](https://vuejs.org/) are very popular choices.

If you're feeling brave and would like to try out a different language, I'd recommend taking a look at either [Elm](https://elm-lang.org/) or ClojureScript's [re-frame](https://github.com/Day8/re-frame).

## Wrapping Up

Before I close, I'd just like to point out that it hasn't been my intention to disparage the React devs. As I mentioned above, I think hooks are a good innovation in many ways, and in most cases an improvement on the class API. I hope I've made it clear that what I'm against is the _overuse_ of hooks, and the belief that they will solve all your problems.

Thanks very much for reading. Although I imagine many people won't agree, I hope I've made some valuable points. If you have any feedback on this post, especially reasons I'm incorrect, I'd love to hear it. Please drop me an email at [mail@davemartin.me](mailto:mail@davemartin.me). If I've piqued your interest, here are some other resources you may find useful:

_Official Docs_

* [Introducing Hooks](https://reactjs.org/docs/hooks-intro.html)
* [Hooks API Reference](https://reactjs.org/docs/hooks-reference.html)
* [Flux: An Application Architecture for React](https://reactjs.org/blog/2014/05/06/flux.html)

_Praise for Hooks_

* [Thinking in React Hooks](https://wattenberger.com/blog/react-hooks)
* [Making Sense of React Hooks](https://dev.to/dan_abramov/making-sense-of-react-hooks-2eib)
* [React Hooks and its Advantages](https://www.boardinfinity.com/blog/react-hooks-and-its-advantages/)
* [React Hooks and Why You Should Use Them](https://medium.com/geekculture/react-hooks-and-why-you-should-use-them-ab92ee033e43)
* [Why React Hooks?](https://ui.dev/why-react-hooks)
* [Why Hooks are the Best Thing to happen to React](https://stackoverflow.blog/2021/10/20/why-hooks-are-the-best-thing-to-happen-to-react/)
* [Replace Redux with React Hooks](https://dev.to/carriepascale/replace-redux-with-react-hooks-o8c)

_Criticisms of Hooks_

* [The Ugly Side of Hooks](https://medium.com/swlh/the-ugly-side-of-hooks-584f0f8136b6)
* [Why the Hate for React Hooks](https://www.reddit.com/r/reactjs/comments/9suobg/why_the_hate_for_react_hooks/) (Reddit post)
* [Why Hooks are Bad](https://leocode.com/development/why-hooks-are-bad/)
* [A Critique of React Hooks](https://dillonshook.com/a-critique-of-react-hooks/)
* [Can we all just admit React Hooks were a Bad Idea?](https://medium.com/codex/can-we-all-just-admit-react-hooks-were-a-bad-idea-c48120c5188d)
* [Hooks Considered Harmful](https://labs.factorialhr.com/posts/hooks-considered-harmful)