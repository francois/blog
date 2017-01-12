---
layout: post
title:  "Ember.js Gotchas from an Experienced Developer, But Raw Beginner at Ember"
date:   2017-01-10
categories: emberjs
---
[Ember.js](https://emberjs.com/) has been intriguing me for a long time. Ember promises to help when building ambitious web applications.

Starting up with Ember was easy enough, since I was green-fielding a new project. Speaking of project, the project I'm building is a sleep tracker. Conceptually, it's rather simple: every night, before sleeping, I hit "start sleeping". When I wake up, I hit "woke up". The system records start and end times, and generates basic stats. If you want to see a Ruby version of the same project, you can visit the [nofrills-sleep-tracker.com](https://app.nofrills-sleep-tracker.com/).

This post holds a few gotchas I encountered while building the application. Incidentally, the Ember.js application is open-source and can be viewed at [francois/nofrills-sleep-tracker-emberjs](https://github.com/francois/nofrills-sleep-tracker-emberjs).


## You cannot call a function on the controller or model from a template

This was the most surprising thing I found while learning Ember. I expected the following code to work as-is:

{% highlight javascript linenos %}
// app/controllers/index.js
export default Ember.Controller.extend({
  isNapping: function() {
    return this.get('state') === 'napping';
  },
})
{% endhighlight %}

{% highlight handlebars linenos %}
{% raw %}
// app/templates/index.hbs
<div class="small-12 columns">
  <p>napping? {{isNapping}}</p>
</div>
{% endraw %}
{% endhighlight %}

When viewed on [Ember Twiddle](https://ember-twiddle.com/2596daefabd8a1e2d3439465b6f0df45), the function's source is printed instead of the function being called. Over on Stack Overflow, my question, [Computed properties in controller in Ember](http://stackoverflow.com/a/41517212/7355), received a good answer. I knew I needed a computed property, but I didn't know that I couldn't call a function. I expected the function to always be evaluated on every render.

Even with the comments I received on the Stack Overflow question, I am still unclear why the function is not applied. The Handlebars template is probably expecting a specific API, which a plain function return value does not provide. Only `Ember.computed` returns the correct API.


## It is better to represent state in the URL instead of keeping state in-memory, especially during development/live-reload

The application has maybe a dozen states total: on the index screen, on settings, on analytics, sleeping and napping. Given that index is the primary entry point, I initially built my like this:

{% highlight handlebars linenos %}
{% raw %}
<!-- app/templates/index.hbs -->
{{#if isSleeping}}
  {{state-sleeping
    startAt=startAt
    timezone=timezone
    didAwaken=(action "didAwaken" "night")
    didCancel=(action "didReset")
  }}
{{else if isNapping}}
  {{state-napping
    startAt=startAt
    timezone=timezone
    didAwaken=(action "didAwaken" "nap")
    didCancel=(action "didReset")
  }}
{{else}}
  {{state-awake
    events=model
    didStartSleeping=(action "didStartSleeping")
    didStartNapping=(action "didStartNapping")
    didAwaken=(action "didAwaken")
  }}
{{/if}}

<!-- app/templates/components/state-napping.hbs -->
<button class="button small" {{action "didAwaken"}}>Awaken</button>
{% endraw %}
{% endhighlight %}

{% highlight javascript linenos %}
// app/controllers/index.js
export default Ember.Controller.extend({
  isNapping: Ember.computed('state', function() {
    return this.get('state') === 'napping';
  }),

  isSleeping: Ember.computed('state', function() {
    return this.get('state') === 'sleeping';
  }),

  actions: {
    didStartNapping: function() {
      this.set('state', 'napping');
      this.set('startAt', new Date());
    },

    didStartSleeping: function() {
      this.set('state', 'sleeping');
      this.set('startAt', new Date());
    },

    didAwaken: function(type) {
      let startAt = this.get('startAt').getTime();
      let endAt = new Date().getTime();

      let newEvent = {
        type: type,
        startAt: startAt,
        endAt: endAt,
      };

      this.get('store').createRecord('event', newEvent).save();

      this.set('state', 'awake');
      this.set('startAt', undefined);
    },

    didReset: function() {
      this.set('state', 'awake');
      this.set('startAt', undefined);
    },
  }
});
{% endhighlight %}

Unfortunately, this may be problematic: in development, when the app live-reloads, all state is lost, because a new instance of the controller is instantiated, and the new instance doesn't receive values from the previous instance. If instead there are multiple URLs, on reload, Ember.js will instantiate the correct controller.

What I'm unclear on yet is when running in production. In production, the app will always start from `/`. The controller there should read the current app state and change the URL to reflect that state, probably using [`replaceWith()`](https://guides.emberjs.com/v2.10.0/routing/redirection/).

Note that I'm advocating, for this specific application, URLs of the form:

* `/`: awake state
* `/:state/:start_at`: sleeping or napping state, with the exact instant where the user said they started sleeping


## Will I stick with Ember?

Frankly, I don't know. I wanted to learn more about Ember. The app I chose to implement is rather small. For a "real" project, it would depend on the exact forces at play and the team's experience with any existing framework. As a team lead, I would prefer to go with something the team already knows, unless a prototype showed major gains over something we already know.

I am still very new at Ember. I like the framework and think it's better than React, initially. The fact that Ember has taken many decisions, is very opinionated, makes it an easier framework to start with, in my opinion.

On the other hand, I prefer React's way of handling state, where there is a distinction between local state and received props. It makes it easier to reason about the data flow. In Ember, I find it harder to determine where a value came from: was it set locally, or was it received from elsewhere?

I also dabbled with [Elm](http://elm-lang.org/) in 2016, and I really liked it. The Haskell roots and the [Elm Architecture](https://guide.elm-lang.org/architecture/) made it easy, for me, to understand.

As usual, there are no universal truths: take your own decisions by prototyping your app using one or more frameworks. You may be surprised with the results!
