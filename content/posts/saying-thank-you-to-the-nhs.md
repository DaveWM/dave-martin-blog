+++
date = 2020-04-22T23:00:00Z
description = ""
draft = true
title = "Saying \"Thank You\" to the NHS"

+++
### (a.k.a how to generate a static site using Clojure)

As many (all?) of you are no doubt aware, during these troubling times many young people are showing their support for the NHS and other key workers by [displaying paintings of rainbows in their windows](https://www.independent.co.uk/life-style/coronavirus-nhs-nightingale-hospital-rainbows-art-drawing-painting-a9452806.html). Since I'm an adult (legally, if not mentally), I thought I'd try to go one better - displaying a rainbow on a monitor.

Naturally, I turned to Clojure to do this. The basic plan was this:

* Use Clojure to generate an HTML page, containing an SVG rainbow
* Host the HTML on Netlify
* Display the site on a monitor using a Raspberry Pi

### Step 1: Generating the HTML

Generating HTML in Clojure is dead easy, thanks to the excellent [Hiccup](https://github.com/weavejester/hiccup) library. Hiccup allows you to take a Clojure data structure, like `[:span "Hello!"]`, and turn it into an HTML string. Our task then becomes writing a Clojure function that returns an SVG rainbow as hiccup. The SVG is composed of concentic circles of each colour, plus a mask to make the bit under the arch transparent. The Clojure code looks like this:

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
         rainbow
         ]]])

Now we just have to render the HTML, and spit it out to a file:

    (defn main [& args]
      (println "Building...")
      (spit "public/index.html" (h/html page)))

That's all we need! We're now ready to get our site set up in Netlify.

### Step 2: Hosting in Netlify

[Netlify](https://www.netlify.com/) is an absolutely fantastic platform for building and hosting static sites or SPAs. It makes it extremely quick to get a site up and running. We basically want Netlify to look at our git repo, and on every commit run our Clojure code then publish the `/public` directory as a website. If you'd like to give this a go yourself, follow these steps:

1. Sign up on [Netlify](https://www.netlify.com/)
2. Fork my git repo [https://github.com/DaveWM/thank-you-nhs](https://github.com/DaveWM/thank-you-nhs "https://github.com/DaveWM/thank-you-nhs")
3. Create a new site in Netlify from your fork of the repo
4. That's all! Netlify will build and deploy the site

"How did Netlify know what to do?" I hear you ask. The magic lies in the `netlify.toml` file in the root of the repo. This gives Netlify a command to run to build your site. The file looks something like this:

    [build]
      command = "lein run"
      publish = "public"

Pretty easy! You can find my site at [https://relaxed-bartik-f774e5.netlify.app/](https://relaxed-bartik-f774e5.netlify.app/ "https://relaxed-bartik-f774e5.netlify.app/").

### Step 3: Displaying the site

To display the site, I dug out an old Raspberry Pi, and hooked it up to a monitor. I then pointed the Pi's browser at my site and made it full screen. Here's the result:

![](/thank-you-nhs.jpg)

Nice! I hope you enjoyed this post. If you'd like to do something to help the NHS in this time of crisis, you can donate to the "Clap for Carers" campaign [here](https://uk.virginmoneygiving.com/ClapForOurCarers).