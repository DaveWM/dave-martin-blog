+++
date = 2020-04-29T23:00:00Z
description = "A tutorial describing how to render HTML in Clojure, and set up a simple static site in Netlify."
title = "Saying \"Thank You\" to the NHS, using Clojure"

+++
### (a.k.a how to generate a static site using Clojure)

As many of you are no doubt aware, during these troubling times many young people are showing their support for the NHS and other key workers by [displaying paintings of rainbows in their windows](https://www.independent.co.uk/life-style/coronavirus-nhs-nightingale-hospital-rainbows-art-drawing-painting-a9452806.html). Since I'm supposedly a grown-up, I thought I'd try to go one better - displaying a computer-generated rainbow. Naturally, I turned to Clojure to do this. My basic plan was this:

* Use Clojure to generate an HTML page, containing an [SVG](https://developer.mozilla.org/en-US/docs/Web/SVG) rainbow
* Host this HTML page on Netlify
* Display the site on a monitor using a Raspberry Pi

In this post, I'll cover how I built and deployed my simple static site. If you'd like to follow along, clone my repo at [https://github.com/DaveWM/thank-you-nhs](https://github.com/DaveWM/thank-you-nhs "https://github.com/DaveWM/thank-you-nhs").

### Step 1: Generating the HTML

Generating HTML in Clojure is dead easy, thanks to the excellent [Hiccup](https://github.com/weavejester/hiccup) library. Hiccup allows you to take a Clojure data structure, like `[:span "Hello!"]`, and turn it into an HTML string. Our task is then to write a Clojure function that returns hiccup representing an SVG rainbow. This SVG is composed of multiple concentic circle, one for each colour of the rainbow, plus a [mask](https://developer.mozilla.org/en-US/docs/Web/SVG/Element/mask) to "cut out" the section under the arch. The Clojure code looks like this:

    (def rainbow
      (let [colours ["red" "orange" "yellow" "green" "blue" "indigo" "violet"]
            rainbow-height 15
            r-step (int (/ (- 50 rainbow-height) (inc (count colours))))]
        [:div.rainbow
         [:svg
          {:viewBox "0 0 100 50"
           :preserveAspectRatio "xMidYMax slice"
           :width "auto"}
          [:defs
           [:mask {:id "hole"}
            [:rect {:width "100%" :height "100%" :fill "white"}]
            [:circle {:cx 50 :cy 50 :r (* 1.5 rainbow-height) :fill "black"}]]]
          (->> (concat
                [:g {:mask "url(#hole)"}]
                (->> colours
                     (map-indexed
                      (fn [idx colour]
                        [:circle {:cx 50 :cy 50 :r (- 50 (* r-step idx)) :fill colour}]))))
               (into []))]]))

This generates an SVG that looks like this:

![](/rainbow.png)

Great! Now we just have to create some hiccup for the entire page, which contains the rainbow SVG. Here's a slightly simplified version of the code ([full code here](https://bit.ly/3bzRt0j)):

    (def page
      [:html
       [:body
        [:div.main
         [:h1 "Thank you NHS!"]
         rainbow]]])

Now we just have to render the HTML, and spit it out to a file:

    (defn main [& args]
      (println "Building...")
      (->> (h/html page) ;; convert our Hiccup to a string
           (spit "public/index.html") ;; spit the string out to an HTML file
     )

_Note: in our `project.clj` we have the line `:main thank-you-nhs.core/main`, which tells `lein run` to run this `main` function._

That's all we need! If you're following along, create a GitHub, GitLab or BitBucket repo, and push the code to it. We're now ready to get our site set up in Netlify.

### Step 2: Hosting in Netlify

[Netlify](https://www.netlify.com/) is an absolutely fantastic platform for building and hosting static sites or SPAs. It makes it extremely quick to get a site up and running. We want Netlify to watch your git repo, and on every new commit run `lein run` then host the `/public` directory as a website. If you're following along and you'd like to give this a go yourself, follow these steps:

1. Sign up on [Netlify](https://www.netlify.com/)
2. [Create a new site](https://app.netlify.com/start) from your repo
3. That's all! Netlify will build and deploy the site and give you a URL for it

"How did Netlify know what to do?" I hear you ask. The magic lies in the `netlify.toml` file in the root of the repo. This gives Netlify a command to run to build your site. The file looks something like this:

    [build]
      command = "lein run"
      publish = "public"

Pretty easy! You can find my site at [https://relaxed-bartik-f774e5.netlify.app/](https://relaxed-bartik-f774e5.netlify.app/ "https://relaxed-bartik-f774e5.netlify.app/").

### Step 3: Displaying the site

To display the site, I dug out an old Raspberry Pi, and hooked it up to a monitor. I then pointed the Pi's browser at my site and made it full screen. Here's the result:

![](/thank-you-nhs.jpg)

Nice! I hope you enjoyed this post. If you'd like to do something to help the NHS in this time of crisis, you can donate to the "Clap for Carers" campaign [here](https://uk.virginmoneygiving.com/ClapForOurCarers).