---
layout: post
title:  "A fast and easy particle system using GPU instancing"
date:   2016-04-11 17:43:45 +1100
categories: graphics c# monogame
blurb: "Render a chaotic galaxy using a single quad!"
preview: /images/posts/particle_preview.png
twitter_blurb: "Use #MonoGame to render a chaotic galaxy using a single quad!"
twitter_image: /images/posts/particle_preview.png
comments: true
disqus_identifier: fast_particle_sys
---

These days, pretty much any game engine comes bundled with some form of *particle system* that handles
 the creation, movement and rendering of a high-number of (typically) small, homogeneous objects within a scene.

![](/images/posts/fire.png)
<em id="caption">An example of a simple particle system simulating fire</em>

When working on [CocosSharp](https://github.com/mono/CocosSharp), we quickly realised that our particle system
performance wasn't ideal &mdash; something that was particularly noticeable on mobile platforms.
Digging around at the implementation, I thought one could redesign things to allow for *GPU-based
instancing* (we'll get to this in a bit) to significantly reduce the CPU-overhead and thereby boosting the 
overall rendering performance.

While an instancing-based particle system is by no means a novel idea, I found it pretty tough to find any accessible resources
online that described the nitty-gritty of such an implementation (at least the specific approach that I have in mind).
So the motivation for this guide is to, at the very least, make developers aware of such an approach in the 
*instance* (that's a pun) that they're seeking to squeeze more performance out of their particle rendering.

> If you've got a good understanding of particle systems and simply want to see the code, then skip to
the [implementation](#implementation)

* TOC
{:toc}

## Before we get started

* While the sample implementation uses C# and [MonoGame 3.5](http://www.monogame.net/2016/03/17/monogame-3-5/), the aim is to make
the guide platform-agnostic, so hopefully it should be equally relevant regardless of your chosen language and tools.

* This is by no means *the* definitive approach for fast particle rendering. At the conclusion, I'll discuss the limitations and advantages relative to other
popular techniques.

## The standard (and slow) implementation

When designing a particle system it is important to keep in mind certain unique characteristics that distinguish them from other renderable objects &mdash; namely,

* we're usually rendering thousands of particles concurrently
* even within a three-dimensional scene, particles are typically rendered as flat *sprites* &mdash; that is we geometrically represent a particle as a single quad
* generally homogeneous in that both the size, colour or underlying texture is consistent (we'll see later how one add some variability)
* the (collision-free) movement path of particles is generally deterministic with small random pertubations to give the illusion of a chaotic system
* collisions between particles is usually ignored

Again, the above list is by no means a strict requirement, but rather features of a particle system's typical use cases (particularly for game development). 

![](/images/posts/stars.png)
<em id="caption">Using sprites to represent each individual particle</em>

### The life of a particle

Additionally, any given particle is usually only rendered for a short, finite time and so any particle system implementation 
typically employs the following process

1. We emit a particle
2. For every update step, we update the particle transform (position, scale, rotation etc.)
3. The particle is rendered with the new transform
4. After a fixed time *t* the particle is destroyed and removed from the scene
4. We continue emitting further particles up to some particle or time limit

> Note, that obviously we can emit new particles while older particles are still alive &mdash; otherwise, it would 
be a pretty boring particle system!

### Laying out our data

Now that we understand the particle lifecycle, a natural approach to setting up our data is
to associate each particle with an *affine transform* that is represented as a 
4x4 (floating-point) matrix and encapsulates all the information about the current position, rotation and scale.
So we would have an array of transforms for each particle. i.e.

{% highlight csharp %}
Matrix[] particleTransforms;
{% endhighlight %}

We know that the number of particles being rendered is dynamic over time so the question is how large should we
make this array? 

> A cornerstone of high perfomance game development (or otherwise) is to keep 
dynamic memory allocation to a minimum.

Rather than constantly rejigging the size of this array to correspond to the number of active particles,
we instead allocate the array *once* for the *maximal* number of particles that may be visible.
For instance, if we have a particle life of 2 seconds and an emittance rate of 1000 particles/second, then 
we require an array of size 2000.

Finally, we need to layout our rendering data &mdash; that is, the *vertex buffer* that will be
passed on to the GPU. Remember, that we're assuming that a particle is rendered as a flat quad, meaning that
we will require 4 vertices per quad.

> Each vertex typically packs the associated position, texture coordinate and colour - a total of 6 bytes of data per vertex
and hence 24 bytes per particle. Moreover, each particle transform is a 4x4 floating-point matrix meaning we have 64 associate bytes per particle.
This may not seem like much, but once we scale the system to render hundreds of thousands or millions of particles, the required amount of data
starts to add up! 

### The CPU update-GPU render cycle

Once familiar with the standard design and data involved with creating
a particle system, the actual computational steps are relatively straight-forward.
First, for a given game we setup a run-loop that will periodically
request the particle system to perform the following

> In MonoGame, a run-loop is automatically setup for you within a `Game` class instance,
periodically calling the `Update` and `Draw` methods

1. Compute each particle's new transform
2. Update the corresponding vertex buffer
3. Load vertex buffer onto GPU
4. Render

![](/images/posts/particle_flow.png)
<em id="caption">Schematic representation of the CPU-GPU render cycle
for a particle system. Note, that a lot of the work is performed on the CPU.</em>


Hopefully, one can see that the bulk of the work is performed by the CPU and moreover there's the further cost of
constantly transferring a potentially large vertex buffer over to the GPU to render. If the particle system was the 
sole element of your scene then this may not be too big of a problem, but once you start adding other
renderable objects, other CPU computations (e.g. physics calculations), input handling etc.; lessening the 
burden on the CPU can dramatically improve the overall performance of your game.


## Using instancing to speed things up

Simply put, *geometry instancing* provides the opportunity to repeatedly
render the same vertex buffer on the GPU within the *same* draw call. More importantly,
we can further include a corresponding *instance buffer* to 
specify attributes that are unique to each instance of the rendered objects.
For example, a common use of the buffer would be to contain the affine
transform matrices of each instance.

![](/images/posts/instancing_flow.png)
<em id="caption">A schematic representation showcasing the steps involved when 
performing geometry instancing on the GPU.</em>
 
In this way, we can repeatedely render a complex geometric object in different positions, scales, rotations 
(and more) within the scene using a single draw call.

### There's instancing and then there's *instancing*

The problem with the standard approach of creating a instance buffer
filled with transforms is that we're still doing a lot 
of the heavy computation on the CPU. Moreover, we still need to 
transfer this large buffer consisting of matrices over to the GPU.

Instead, while we still will employ geometry instancing for our solution,
the approach we will take is to first slim down the data
of our instance buffer to simply contain the *starting time* of a particle.
Then, in the upcoming section, we show how we can 
perform essentially *all* our computations related to updating a particle's state on the GPU.

![](/images/posts/instancing_slim_flow.png)
<em id="caption">The proposed alternate instancing implementation. Notice how we're highlighting that the 
passed-in instance buffer is much slimmer, which will then be subsequently expanded on the GPU.</em>

## Implementation

Now that the theory is behind us, let's get to coding! In my case, I kick-start
a new MonoGame Visual Studio project that provides the skeleton code for a game.
In particular, an instance of the `Game` class will initiliase our content and it's
from here that the game's run-loop calls `Update` and `Draw` are called.

> For more details on getting started with MonoGame check out this great post 
[here](https://blog.xamarin.com/build-your-first-game-with-monogame-getting-started/)

We're going to be encapsulating the functionality of our particle system within a 
`FastParticleSystem` class. To begin we define the properties of our system

{% highlight csharp %}
class FastParticleSystem
{
	// Core properties every particle system needs
	
	// The dimensions of each particle sprite
	public Vector2 ParticleSize { get; set; }
	
	// The corresponding texture used
	// when rendering the sprite
	public Texture2D ParticleTexture { get; set; }

	// The life of a particle in seconds
	public float Life { get; set; }
	
	// The number of particles that are created
	// every second
	public uint EmitRate { get; set; }

	// ... More properties (see later)
}
{% endhighlight %}

Once a user has created and customised their particle system, 
we call `RefreshParticleSystemBuffers` on the instance to generate
both the vertex and instance buffers. In the former case, the vertex buffer
is a very small one consisting of just four vertices &mdash; a single quad

{% highlight csharp %}
public void RefreshParticleSystemBuffers(GraphicsDevice graphicsDevice)
{
	// Create a single quad centered at the origin
	float halfWidth = ParticleSize.X / 2;
	float halfHeight = ParticleSize.Y / 2;

	VertexData[] vertices = new VertexData[4];
	vertices[0].Position = new Vector3(-halfWidth, -halfHeight, 0);
	vertices[1].Position = new Vector3(halfWidth, -halfHeight, 0);
	vertices[2].Position = new Vector3(-halfWidth, halfHeight, 0);
	vertices[3].Position = new Vector3(halfWidth, halfHeight, 0);

	// Our vertex data only includes texture coordinates
	// but you could also include colours (and more)
	vertices[0].TexCoords = new Vector2(0.0f, 0.0f);
	vertices[1].TexCoords = new Vector2(1.0f, 0.0f);
	vertices[2].TexCoords = new Vector2(0.0f, 1.0f);
	vertices[3].TexCoords = new Vector2(1.0f, 1.0f);

	vertexBuffer = new VertexBuffer(graphicsDevice, 
		VertexData.VertexDeclaration, 4, BufferUsage.WriteOnly);
	vertexBuffer.SetData(vertices);
	vertexBufferBinding = new VertexBufferBinding(vertexBuffer);
{% endhighlight %}

There's also an associated *index buffer* whose initialisation I've omitted from the snippet for 
the sake of brevity. 

> In MonoGame, the provided interface for geometry instancing requires the use of indexed-primitives &mdash;
that is, your vertex buffer must have a corresponding index buffer

Next, we create our instance buffer which will store data specific to each particle. 
The key idea is that unlike other approaches that store entire matrix transforms per instance, our buffer
is going to be far simpler - namely, we are solely going to specify the particle starting time 

{% highlight csharp %}
instanceData = new InstanceData[MaxVisibleParticles];
instanceBuffer = new VertexBuffer(graphicsDevice, InstanceData.VertexDeclaration, 
	MaxVisibleParticles, BufferUsage.WriteOnly);

// Key point! We set the final argument to 1 to specify the 
// instance frequency
// This ensures that each "vertex" of the instance buffer 
// corresponds to the four vertices that make up a particle
instanceBufferBinding = new VertexBufferBinding(instanceBuffer, 0, 1);

// Initialise the starting time of each particle
for (int i = 0; i < MaxVisibleParticles; ++i)
	instanceData[i].Time = -(i + 1) / (float)EmitRate;

{% endhighlight %}

Finally, once the buffers are set we render them

{% highlight csharp %}
public void DrawParticles(ref Matrix worldViewProj, GraphicsDevice graphicsDevice)
{
	// Load and setup our shader - see next section
	// ...

	// Load our texture and buffers
	graphicsDevice.Textures[0] = ParticleTexture;
	graphicsDevice.SamplerStates[0] = SamplerState.LinearClamp;
	graphicsDevice.SetVertexBuffers(vertexBufferBinding, instanceBufferBinding);
	graphicsDevice.Indices = indexBuffer;

	graphicsDevice.DrawInstancedPrimitives(PrimitiveType.TriangleList, 0, 0, 6, 0, 2, numInstancesToDraw);
}
{% endhighlight %}

> Warning: As of MonoGame 3.5, `DrawInstancedPrimitives` is only supported by
DirectX targets, however the aim is to provide full cross-platform support.

> Remember, MonoGame is an open-source project and relies on the community
to help improve the framework. If you're interested in contributing, check out the
guide [here](https://github.com/mono/MonoGame/blob/develop/CONTRIBUTING.md)
to get started.

### The particle shader

In the previous section we described how our instance buffer consists
of the starting time of a given particle, but we didn't describe how
this would result in updating the state (position, size, colour etc.)
of a given particle. In short, we will compute this
directly on the GPU via our custom *vertex shader*.

> If you're not familiar with shaders and the *programmable GPU pipeline*, a good introduction
can be found [here](https://www.opengl.org/wiki/Rendering_Pipeline_Overview).

> For MonoGame users, documentation for creating and using custom effects (i.e. shaders)
can be found [here](http://www.monogame.net/documentation/?page=Custom_Effects).


As an example, let's say we want to simply move a particle vertically down (in two-dimensions)
over a distance *d* for a given life-time. We could then parameterise the position of our particle
relative to time &mdash; that, is

$$
x(t) = 0 \\
y(t) = -d \times t 
$$

where *t* is the time normalised to be between 0 and 1 (i.e. 0 corresponds to when the particle is emitted,
 1 corresponds to the full life-time in seconds). Translating this into code, we define
 the shader constants

{% highlight csharp %}
float TotalTime;
float Life;
float Distance;
matrix WorldViewProjection;
{% endhighlight %}

which would be initialised during the `Draw` call i.e.

{% highlight csharp %}
public void DrawParticles(ref Matrix worldViewProj, GraphicsDevice graphicsDevice)
{
	particlesEffect.CurrentTechnique = particlesEffect.Techniques["ParticleDrawing"];

	// Initialise our shader constants
	particlesEffect.Parameters["WorldViewProjection"].SetValue(worldViewProj);
	particlesEffect.Parameters["TotalTime"].SetValue(TotalTime);
	particlesEffect.Parameters["Life"].SetValue(Life);
	particlesEffect.Parameters["Distance"].SetValue(Distance);
	particlesEffect.CurrentTechnique.Passes[0].Apply();

	// .. rest of draw code from previous section
}
{% endhighlight %}

and subsequently within the vertex shader, we compute the current time of the particle and
normalise it to lie between 0 and 1. Finally, we use the current time to fully determine
the position of our given particle

{% highlight csharp %}
float Life;
float TotalTime;
float Distance;

matrix WorldViewProjection;

VSOutput MainVS(in VSVertexInput vertexInput, float pStartingTime : BLENDWEIGHT0)
{
	VSOutput output = (VSOutput)0;

	// Scale the time to be between the interval 0 to 1
	// i.e. Offset starting time with total time and scale by life
	// It's faster to do this computation on the GPU
	float timeFraction = saturate(fmod((pStartingTime + TotalTime) / (float)Life, 1.0f));

	// Update the particle partition in world coordinates
	output.Position = vertexInput.Position;
	output.Position.y += - Distance * timeFraction;

	// Translate the new position to screen coordinates
	output.Position = mul(output.Position, WorldViewProjection);

	// Rest of VS shader related to colour and/or texture etc.
	// ...

	return output;
}
{% endhighlight %}

Once we get the hang of expressing the particle state as a function of time, anything
is possible. For example, 

$$
x(t) = r cos (2 \pi t) \\
y(t) = r sin (2 \pi t)
$$

would give us a circular particle path, where *r* is a constant radius that 
could be passed-in via the shader. Of course, we're not limited to simply parameterising
the position, but can do the same for a particle's colour, rotation, size and more.

### Adding randomness

Typically, a particle system is attempting to simulate a complex, chaotic system,
which can be achieved by incorpoating some randomness into either both the starting or desired ending
state for each distinct particle. For example, considering again a simple particle moving vertically,
we may want to specify a range of *x* staring positions

$$
-5 \leq x(t) \leq 5 \\
y(t) = -d \times t
$$

So we would initialise each particle to randomly select a starting horizontal position witin
this desired range.
To incorpate this into our code, we first update our instance buffer to 
include a random interval between 0 and 1.

{% highlight csharp %}
public void RefreshParticleSystemBuffers(GraphicsDevice graphicsDevice)
{
	// Initialisation of buffers from
	// previous section
	
	Random rnd = new Random();

	for (int i = 0; i < MaxVisibleParticles; ++i)
	{
		instanceData[i].Time = -(i + 1) / (float)EmitRate;

		// Sample a random number between
		// 0 and 1
		instanceData[i].RandomInterval = 
			rnd.Next(0, MaxVisibleParticles + 1) / (float)MaxVisibleParticles);
	}

	// Rest of Initialisation
	// from previous section
}
{% endhighlight %}

We then subsequently update the draw call to specify the range
of starting positions a particle can take

{% highlight csharp %}
public void DrawParticles(ref Matrix worldViewProj, GraphicsDevice graphicsDevice)
{
	// Setup from previous section
	// ...

	particlesEffect.Parameters["xPositionRange"].SetValue(xPositionRange);

	// Draw code seen previously
}
{% endhighlight %}

Finally, we update our vertex shader code to incorporate our randomness

{% highlight csharp %}
VSOutput MainVS(in VSVertexInput vertexInput, float pStartingTime : BLENDWEIGHT0,
float randomInterval : BLENDWEIGHT1)
{
	// Initialisation of VS shader as described in previous section
	// ...
	
	float xPos = lerp(xPositionRange[0], xPositionRange[1], randomInterval);

	// Update the particle partition in world coordinates
	output.Position = vertexInput.Position;
	output.Position.x = xPos;
	output.Position.y += - Distance * timeFraction;

	// Rest of VS shader as described in previous section

	return output;
}

{% endhighlight %}

> In general, it may not be necessary to constantly refresh the instance buffer with
new random variables over time. Aside from there being a performance benefit to avoid this computation, 
if the number of particles rendered is relatively large, refreshing this large array of random variables
may not visually be noticeable.

### Putting it together

Finally, one can also utilise the *instance index* that's passed into the vertex
shader to further add some variability. Specifically, within our shader
we can define our input as

{% highlight csharp %}
struct VSVertexInput
{
	float4 Position : SV_POSITION;
	float2 TexCoord : TEXCOORD0;

	// Attribute that is injected by the GPU
	// We do not provide it
	uint InstanceId : SV_InstanceId;
};
{% endhighlight %}

which allows us to distinguish each instance call via the passed-in `InstanceId`.

Combining all the elements discussed together, one can construct quite complex and 
sophisticated particle systems. For example, I was able to create a very slick looking galaxy by
doing the following

* Emitting particles radially
* Radius is a function of both time as well as the particle id
* Use variable starting and ending colours within a random range
* Colour alpha value is a function of time
* Depth of particles is a function of the particle id to enable depth-sorting

The result:

![](/images/posts/galaxy_particle_loop.gif)
<em id="caption">Galaxy consisting of 200,000 particles!</em>

Again, your particle system doesn't necessarily have to be this complex, but
is more a highlight of what is possible.

## Is it worth it? Let me work it...out

Despite the relatively complex computations performed on the vertex shader within my example, I was able to consistently
render over 400,000 particles (800,000 triangles) at a consistent 60 frames per second on an Integrated GPU (Surface Pro 4), which, personally, I thought
was fantastic given that my experience with [standard particle systems](#the-standard-and-slow-implementation) struggling with numbers less than a hundred-thousand. 
However, there are a few things to keep in mind when you are comparing performance

* **Resolution** Within my example, my game was running at 1024x768. In general, the higher the resolution the lower the frame-rate.
* **Fill-rate** Similarly, the size of your particle sprites affects performance.
* **Complexity of shader computations** Simply translating particle positions versus, for instance, applying trigonometric calculations, colour, collisions etc. 
* **Alpha testing and culling** can also impact the time spent processing each particle

So this isn't to say that faster implementations exist, which we'll discuss in the upcoming section, but simply that we need to
be aware that when making comparisons, we're comparing apples with apples.

## Some limitations and alternative approaches

Both the strength and weakness of this implementation is that we are relying on the state of a particle to be fully determined
as a function of time. It is for this reason we're able to pass-in a slimmed-down instance buffer consisting of particle time (and
random intervals to give the illusion of chaos) and then rely on the GPU to determine all the properties of the system.
As soon as a particle's state can't be solely determined ahead of time (such as when incorporating collision effects), then this
approach no longer becomes as viable.

Additionally, with respect to collision-free particle system's there may be a few additional changes
that we could have made to further help performance

1. **Use a geometry shader** In our approach, the vertex shader is called a total of four times for any given particle, resulting
in a lot of repeated computations, such as determining the particle time. As an alternate approach, we could ask the vertex shader to render *points* rather than quads, which
would subsequently be expanded into quads by the geometry shader. This would avoid repeated computations because a point would be uniquely mapped to a particle.
As an additional benefit, the geometry shader gives us the opportunity to discard particles that aren't visible, potentially improving performance.

2. **Use a compute shader** I have come across users speeding up their particle systems by employing *compute shaders* &mdash;
a shader stage for performing arbitrary computations on the GPU. The advertised benefit is that we are off-loading the heavy computations of determining
particle state to the compute shader, leaving the vertex and fragment shaders to solely perform the rendering, resulting in a more streamlined pipeline. It would be
interesting to see what kind of gains could be achieved with this approach.

> As of MonoGame 3.5, both geometry and compute shaders are unsupported.

## Wrapping up and sample project

I hope you enjoyed this guide and that I've convinced you that, at the very least,
there are better alternatives to the standard CPU update-GPU render approach when designing a 
particle system.

The sample project demonstrating my funky galaxy system can be found [here](TBA), and if you  have any 
feedback or suggestions please let me know down in the comments. Thanks!


