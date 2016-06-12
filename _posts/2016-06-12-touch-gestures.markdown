---
layout: post
title: "Create custom touch gestures for mobile games and apps"
date: 2016-06-12 17:43:45 +1100
categories: touch machine-learning defold
blurb: "A simple application of machine-learning to create a more responsive mobile user-experience"
preview: /images/posts/gesture_preview.png
twitter_blurb: "A #defold module using #machine-learning to create custom touch gestures"
twitter_image: /images/posts/gesture_preview.png
comments: true
disqus_identifier: custom_gestures
---

Personally, I feel the use of *custom* touch gestures within mobile apps, particularly mobile games, is used too sparingly.
While all the major mobile APIs (iOS, Android, WindowsPhone) provide functionality for detecting a host of
predefined gestures, in general, the current API approach for defining custom touch gestures is not as mature.
For example, iOS provides the bare-bones abstraction `UIGeometryRecognizer` for defining custom gestures, but simply describes
a common interface that must be implmented and fully relies on the developer to identify when a sequence
of touch events corresponds to a given gesture. The result is that developer's have typically relied on a geometry-driven approach
to identify custom gestures --- that is, given the position, angles, distance etc. between a series of touches,
can we identify when the gesture is performed.

![](/images/posts/platformer.png)
<em id="caption">Mobile platformer with traditional inputs overlaid. Using custom gestures we can create a better user experience!</em>

Instead, I want to implement a *data-driven* approach to the problem of detecting custom touch gestures. In particular, by
a simple application of machine-learning, we'll see that we end up with a solution that is far more robust and extensible.

# Before we start

While this guide is meant to be platform-agnostic, I'll be coding things up using [Defold](http://www.defold.com/) 
&mdash; a free and publicly available game-engine created by [King](https://king.com/). In particular,
with *Defold*

* games are created with the bundled game-editor
* it is cross-platform (iOS, Android, Windows, MacOS X, HTML5 and Linux)
* uses [Lua](https://www.lua.org/) to interface with the API
* has a particular focus on 2d-games

![](/images/posts/defold.png)

The motivation for using *Defold* was purely educational &mdash; I thought this project was a great way of testing out this new
game-engine to see what works well and what could use some improvement. But for this guide I'll be avoiding giving any explicit feedback,
and focus exclusively on the project itself.


# Implementation

For newcomers, machine-learning can appear to be quite a nebulous and intimidating domain, but not every
application of machine-learning has to reach the same lofty technical heights as Google's [AlphaGo](https://deepmind.com/alpha-go).
Here, we're going to be solving arguably the most classical problem within machine-learning &mdash; that is,
distance-based classification using supervised training data. That may be a mouthful, but as we'll see, in practice, 
the approach is quite straight-forward.

## 1. Store labelled training data for each custom gesture

The first step is to collect a training set for all the custom touch gestures that we're interested in classifying. Specifically,
we want to construct a dictionary mapping gestures to all the corresponding series of touch positions over time that make up our gesture
as highlighted in the figure below:

![](/images/posts/training_data.png)
<em id="caption">Schematic representation of how we store training data for a given custom gesture</em>

> Keep in mind that we're storing our series of touches relative to the *starting* touch position of the gesture, because we're interested in
the identifying the relative geometry the path of touches forms, not where on the screen this occurred. In other words, if for example a user
flicks-up on the screen, we should be able to identify this gesture irrespective of where on the screen this gesture was performed.


## 2. Find the "typical" representative of each gesture

Once equipped with our training data, the next step is to construct the *exemplar* &mdash;
the ideal representative that best captures the characteristics for each of our gestures. For our purposes, each
exemplar is found by taking the *point-wise average* of the corresponding series of touches. As an example, suppose
for a given gesture we have the contrived training set consisting of three touch paths *p*, *q* and *r* with only three
time points

$$
p = \{ (0,0), (1,0), (1,1) \}, \\
q = \{ (0,0), (2,0), (2,1) \}, \\
r = \{ (0,0), (-1,0), (2,2) \} \\
$$

then we would define the exemplar *E* to be

$$
E = \frac{1}{3} \times \{ (0,0), (1+2-1,0), (1+2+2,1+1+2) \} \\
= \{ (0,0), (2/3,0), (5/3,4/3) \} 
$$

> In general, how one defines the exemplar is dependent on the particular problem context and the form of the training data. While
taking the point-wise mean may appear to be the most natural approach one does need to formalise this to justify its appropiateness. 
In particular, one can show that this exemplar minimises the Euclidean distance between a set of touch-paths and 
hence provides us with a good *geometric* fit of the data.

> Note: A slight nuance when constructing our exemplars is that our training touch-paths may not all be of uniform length as training
data obtained from manual user input is almost certaininly going to have variability. i.e. Is the user going to replicate the exact some touch-path
perfectly each time? To handle this, we prune each of our touch-paths to the *minimal* number of time-points.

![](/images/posts/training_interface.png)
<em id="caption">The training interface for our sample project. Here, we create the training data that is used to build the exemplar for each gesture</em>

## 3. Apply the model to classify incoming touches

Finally, utilising our exemplars, we can classify incoming touch-paths to the closest custom-gesture. Here, our measure of closeness
is considered to be the Euclidean distance between the input touch-path $$p$$ and the set of exemplars $$g_1, g_2, \dots, g_n$$. Formally,
we want

$$
\text{argmin}_{i} \| p - g_i \|.
$$

So in words, given an input touch-path we want to output the index belonging to the corresponding closest custom gesture exemplar.

> In practice, an input touch-path may not be a good fit for any of the available custom gestures that are defined by the given training
data. One way to handle this is to additionally require a minimum distance threshold that the closest exemplar must fall under. If the 
distance is too large, we simply don't classify the input.

## Putting it together

In code, we start a new Defold project as outlined [here](http://www.defold.com/tutorials/getting-started/) and create a new module `custom_gestures.lua`.
Here, modules allow us to define reusable functionality that can be shared across multiple objects and projects &mdash; see the [Defold manual](http://www.defold.com/manuals/modules/) for more information.
Specifically, we specify a `create` function that returns a gesture controller instance that encapsulates our custom gesture training data

{% highlight lua %}
--- Module for defining custom gestures
-- @module custom_gestures
local M = {}

--- A fixed constant outline the minimum number of time points
--- needed to classify a gesture. We could make this configurable
-- @field MIN_PATH_POINTS
local M.MIN_PATH_POINTS = 30

--- Creates a specific instance of the gesture controller to be
--- passed in to module functions.
-- @return Instance of gesture_controller
function M.create(gest_types)
    local gesture_controller = 
    {
        gesture_types = gest_types,
        training_data = {}
    }
    return gesture_controller
end
{% endhighlight %}

To then use this module we would then call the following within a given Defold game object script

{% highlight lua %}
custom_gestures = require "modules.custom_gestures.lua"
--- The fixed key-value list of custom-gestures
local gesture_types = {hash("jump"), hash("slide"), hash("attack")}
local gesture_controller = custom_gestures.create(gesture_types)
{% endhighlight %}

Note, that the above snippet assumes the module is located in the root `modules` subfolder as seen below

![](/images/posts/defold_module_proj.png)
<em id="caption">Sample project directory to load custom module</em>

Loading our training data is a pretty standard affair

{% highlight lua %}
function M.add_training_path(gesture_controller, gesture_hash, training_path)
    table.insert(gesture_controller.training_data[gesture_hash].training_paths, training_path);
end
{% endhighlight %}

and once we've loaded all our data, we want to find the exemplars for each gesture. In our case, this 
corresponds to the point-wise average

{% highlight lua %}
function M.compute_mean_paths(gesture_controller)
    for k, gesture in ipairs(gesture_controller.gesture_types) do
        gesture_controller.training_data[gesture].avg_training_path = {};
        
        local min_path_points = M.MIN_PATH_POINTS;
        
        --- Find the smallest length path in our training data
        --- and prune all other training data accordingly
        for v, path in ipairs(gesture_controller.training_data[gesture].training_paths) do
            min_path_points = math.min(min_path_points, table.getn(path));
        end
        
        for v=1,min_path_points do
            table.insert(gesture_controller.training_data[gesture].avg_training_path, vmath.vector3(0,0,0))
        end

        local n_paths = table.getn(gesture_controller.training_data[gesture].training_paths);
                
        for v, path in ipairs(gesture_controller.training_data[gesture].training_paths) do
            if n_paths > 0 then	
                for x, point in ipairs(path) do
                    if x > min_path_points then
                        break
                    end
                    
                    gesture_controller.training_data[gesture].avg_training_path[x] = 
                        gesture_controller.training_data[gesture].avg_training_path[x] + point;
                end
            end
        end
        
        --- Finally, take the point-wise mean of our exemplars
        for i=1,min_path_points do
            gesture_controller.training_data[gesture].avg_training_path[i] 
                = gesture_controller.training_data[gesture].avg_training_path[i] * (1/n_paths);
        end
    end
end
{% endhighlight %}

Finally, once we have our exemplars, we can perform the classification

{% highlight lua %}
function M.find_closest_gesture(gesture_controller, test_path)
    local closest_gesture = nil
    local closest_dist = 1e10
    
    for k, gesture in ipairs(gesture_types) do
        local avg_path = training_data[gesture].avg_training_path
        local min_points = math.min(table.getn(test_path), 
            table.getn(avg_path));
        dist = 0;
        
        for k=1,min_points do
            dist = dist + vmath.dot(test_path[k] - avg_path[k], test_path[k] - avg_path[k])
        end

        if dist < closest_dist then
            closest_dist = dist;
            closest_gesture = gesture;
        end
        
    end

end
{% endhighlight %}

# A problem &mdash; latency

An issue we encounter when attempting to classify touch gestures is that of latency. If we define quite an expansive custom touch-gesture
that consists of a relatively large number of time-points, then naively we would have to wait for this entire input to be completed before we can classify the gesture.
That delay in user-response, particularly when considering mobile games, can be problematic.

So one simple (but effective) solution is to attempt to classify early prior to the full input touch-path being provided. Exactly when you attempt
to perform the classification is going to be strongly dependent on the exact characteristic of the custom gestures.

> In general, the more distinct the set of custom-gestures, the less input we require in order to classify our series of touches

This raises a pretty important question: If we can fully determine an incoming gesture with partial-information, why not simply define
the gesture as that shortened touch-path? I think there's a few things to keep in mind:

1. **Psychology** I think part of the appeal of custom gestures is there is a more natural correspondence to their underlying actions 
relative to traditional inputs (e.g. flicking your finger upwards to jump versus pressing the A button). This suggests that users will
still tend to complete the full input touch-path even though we're classifying the gesture prematurely.
2. **Gestures don't need to be atomic operations** More importantly, classifying using partial information opens the possibility of 
dynamically changing the response to a given series of touch inputs prior to the full gesture being performed. For example,
imagine within a mobile game being able to alter the height of a jump gesture based on the length of the upward flick across the screen. 

## Wrapping up

Overall, by an application of machine-learning, we're able to implement a simple and robust approach to custom-gesture classification
that can be incorporated within your mobile game or app.

![](/images/posts/custom_gesture.gif)
<em id="caption">The full sample in action &mdash; training and testing!</em>

Keep in mind that one current limitation is that we do not handle multiple touches, but we could certainly extend our approach
to allow for this. In particular, our training data would need to consist of multiple touch-paths.


I hope you enjoyed this guide and please leave any comments or suggestions below. Thanks!



