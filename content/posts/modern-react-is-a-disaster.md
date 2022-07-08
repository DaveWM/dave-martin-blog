+++
date = 2022-07-07T23:00:00Z
description = "Modern React, including hooks and providers, has been a disaster for web development."
draft = true
title = "Modern React is a Disaster"

+++
As you may have guessed from the title, I'm not a big fan of React's hooks API. I've now worked on 3 React applications which heavily use hooks. Each time the code quickly became bloated, difficult to understand, and full of bugs. If you haven't heard of hooks, they're a replacement for React's class API. The hooks API aimed to be more functional, composable, and terser than the class API.

To some extent, it has been successful in this. I'm not *absolutely* against hooks, and I do think they can be used effectively as part of a larger framework. The Redux integration is a good example. However, it's my strong opinion that by themselves hooks are not sufficient to build an application. I'm very much against the, seemingly ubiquitous, belief that hooks obviate any need to architect your application. Nothing could be further from the truth. In this post, I hope to convince you of that.

Firstly, allow me to steelman the argument for hooks. Say you have a `Todo` component, written using the class API: 
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
This is both terser and clearer. You don't need any knowledge about classes, constructors, inheritance, or the `super` keyword. Hooks allow you to write components as a function. You don't have to think about lifecycle methods (`componentDidMount`, `componentDidUpdate`, etc.). Hooks are more composable and reduce repetition.

Unfortunately when you overuse hooks, things quickly go off the rails. Taking our example from above, let's add an input property and a simple side effect: 
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
At first glance that looks great. However, on closer inspection we've introduced a bug! We're reading the `done` property in the `useEffect` function, but we haven't added it to the dependencies array. This may seem trivial to fix, but it can be extremely difficult to track down bugs like this. In my experience the dependencies array is an endless source of bugs. Linters can help, but don't eliminate these bugs. Another "gotcha" occurs if you use anything other than a primitive value as a dependency. This is never mentioned in the documentation. I have spent many hours tracking down these types of bugs, and have no desire to spend any more.

Another major hurdle is the infamous "Rules of Hooks". These approximate to "the number and order of hooks for a given component has to remain constant". In practice, this means that you can't conditionally use a hook. You have to put the conditional in a function, that is passed to the hook. This isn't natural, and can make your code difficult to understand. If you need a variable number of hooks, you have to wrap each one in a child component. This can lead to a proliferation of mostly useless child components, and can disrupt your carefully designed component hierarchy. The root cause of these restrictions is that React tracks hooks by the order in which they are called. I understand the reasoning behind this design choice, but it leads to a lot of problems nonetheless.

Hooks are more composable than classes, but less so than functions. Consider the following component, that makes 2 effects - `foo` and `bar`: 
```js
function useDoSomething(message) {
  useEffect(() => {
   setTimeout(() => {
     console.log(message);
   }, Math.random() * 1000) 
  }, []);
}

function App() {
  useDoSomething("foo");
  useDoSomething("bar");

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
This works, but it's counterintuitive to say the least. Similar code using promises is far more understandable. The reason for this, fundamentally, is that composing hooks is difficult. The only tools you have for composition are state and the dependencies array. While they can be used to craft elegant solutions, in the vast majority of situations they are very clunky.

Unfortunately, that's far from the only drawback of hooks. They also make testing very difficult. Testing is always difficult when you have local state in components. However with the class API, it was often possible to set a component's state then assert that the output of `render` is correct. In general, with hooks this isn't possible. This is because you can use multiple `useState` hooks within a single component. You are forced to cajole the component into the right state, often by simulating a sequence of UI actions. This is time consuming, tedious, error prone, and leads to bloated tests. Also, you are often forced to add delays, which drastically slow down your tests.

The drawbacks I've outlined above are major hinderances, but ones that can in principle be surmounted. If hooks naturally led to a good architecture, it may be worth putting up with these problems. Unfortunately, I don't believe this is the case. To explain why, I'll first have to articulate my thoughts on state management. Managing state is one of the central problems in programming. Over the years, every approach imaginable to UI state management has been tried. 2 broad categories of architectures emerged - object oriented (OO), and functional. OO architectures include MVC/MVVM frameworks like Backbone, Ember, and Angular. It is characterised by storing state locally in components, and allowing it to be freely mutated. OO frameworks provide tools to manage this local state, and synchronise it between components. These include dependency injection, 2-way binding, services/factories, and event buses. Although I do think OOP has its flaws, when done well it does allow you to write relatively clear, testable code. Functional architectures usually look something like the flux architecture (although there are variations, such as Purescript's Halogen). They are characterised by keeping state in as few places outside of components, in as few places as possible. Mutations of this state are tightly controlled. Components are ideally pure functions of `state -> DOM`, which requires extracting side effects to another place in the code. Using this style of architecture makes it easy to write, debug, and test your application. Since pure functions are used as much as possible, your code becomes highly composable with as loose coupling.

As a framework, React is severely lacking. To demonstrate this, let's compare some modern React code to good ol' AngularJS: 
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
angular.module('app').controller("CounterController", function($scope){
  $scope.counter = 0;
  $scope.watch('counter', (val) => console.log(val));
});
```
It doesn't appear that we've made massive strides, at least for this simple example. The AngularJS code is shorter, and more straightforward. AngularJS provided many tools and conveniences that React does not. AngularJS gave you dependency injection, 2-way binding (including with child components), services/factories, and even an event bus in the form of `$emit`. These all helped you constrain the complexity inherent in using local state and side effects. It's also interesting to note that using multiple chained `$scope.$watch`s was always deemed an antipattern. By contrast, hooks both encourage and require this pattern. To be clear, I'm not advocating that Google resurrect AngularJS or that it was strictly better than modern React. My point is that React has come full circle. It now very much resembles the frameworks it sought to supplant, except without a lot of the features that those frameworks provided.

This brings me to the main reason I'm against hooks. When you rely entirely on hooks you inevitably end up with an OO architecture, but without any of the tools you need to manage and synchronise local state. Components can freely make side effects, which makes testing difficult. Dependency injection makes testing feasible, but you don't have this tool. Components have local state, which often needs to be synchronised with the rendered DOM or other components. Synchronising this state is very manual and error prone, because you're not given 2-way binding. To test any component, you need a way of setting its state. You're not given any tools to do this. All this leads to large amounts of complexity, with no way of restraining it. React is one of the most popular libraries in the world, how could I possibly say that it's so woefully insufficient? The answer is that React was never intended as a framework, but solely as a view layer. This is the fundamental reason why using it as a framework is doomed to failure. When it was first released in 2013, React was described as "a library for building composable user interfaces. It encourages the creation of reusable UI components which present data that changes over time." It was designed to be the view layer of your application, never the entire architecture. It was recommended to use the functional Flux architecture with React. React was never designed as a framework. In fact, it was a reaction (pun intended) to JS frameworks becoming increasingly large, burdensome, and difficult to use. Unfortunately, now the prevailing belief in the JS community is that React should be used as a framework. This is misguided.

I've been painting a pretty bleak picture so far. However, the good news is there are solutions. The key is to reduce your reliance on hooks. One option is to use a framework. There are many to choose from, and it's not too difficult to add them to an existing codebase. You can't go far wrong with Redux and Redux Toolkit. It is possible to add OOP features, such as dependency injection, to React. However, in practice it's almost always easier to use an off-the-shelf MVC or MVVM framework. Angular is a great example. If you'd like to try a different language, take a look at Elm or ClojureScript and Re-Frame.

I hope this has given you some food for thought, and caused you to reconsider using hooks. If you have any feedback, please don't hesitate to email me at mail@davemartin.me. Thanks for reading!