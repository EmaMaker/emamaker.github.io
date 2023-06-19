title: Voxel Engine (again)

(Start of the project: August 2022 (java), October 2022 (c++/opengl), ongoing)

<a href="https://git.emamaker.com/emamaker/voxel-engine">Project Repo</a>
<a href="https://github.com/emamaker/voxel-engine">Mirror</a>

<img alt="A landscape" src="/resources/projects/voxel2/landscape.png"/> 

# Making a Voxel Engine

Wait, haven't I already made one? Well yes, I had lots of fun and learned a lot. Maybe it was a bit
too big of a project to choose as one of my first projects, but I did it anyway.

As time passed, and I learned more things about programming, data structures and best pratices, I
started noticing all the errors that I did in the old voxel engine, and the stupid programming
mistakes and overall lack of organization in the code. As I entered university, I also started
learning more about data structures.

Curiosity sparked in me again and I started searching online on
Voxel Engines: which data structures are commonly used, which rendering techniques, lighting,
whether it is better to run stuff mostly on CPU or GPU, how to properly generate procedural terrain?
I was looking most of my information on reddit, mostly on <a
href="https://reddit.com/r/proceduralgeneration">r/proceduralgeneration</a> and <a
href="https://reddit.com/r/VoxelGameDev">r/VoxelGameDev</a>. Subreddit's wikies, showcase posts by users and
especially comments on question posts can be a real gold mine of information.

At this point, I've been also wanting to properly learn C++ and OpenGL for a while, so I started
from that. I've been using <a href="https://learnopengl.org">LearnOpenGL by Joey de Vries</a>, which in turn
suggests <a href="https://learncpp.com">learncpp.com</a> to learn C++. This is what I ended up using for the
current voxel engine. 

My intention is to develop a cubical voxel engine (Minecraft-like if
	you will), [just like last time](/projects/voxelengine.html), trying to be as light as possible (especially on the RAM/VRAM). Ideally, I would love
to run a stripped-down version of this engine on my Nintendo Wii. [(It wouldn't even be the first
	time I try making a game for the wii)](https://github.com/emamaker/a-maze-ing/tree/wii). It
will probably have to very really _really_ stripped down.
Keeping these costrains in mind, I'm also trying to get the highest render distance possible.
(keeping Minecraft as a baseline, I'd say a render distance of 16 chunks of 16x16x16 blocks -which
 is 256 blocks- is a good baseline).

After a lot of research and internal debating, [which you can read about in my
notes](/notes/voxel.html), I ended up with the following "stack":

- C++ and OpenGL
- Divide the world into chunks
    
    - Info about blocks in chunks are stored as an IntervalMap
    - Chunks are stored in a HashMap

- Meshing and generation is done on the CPU using GreedyMeshing and OpenSimplexNoise
- GPU does Phong lighting and texturing
    
I am still using greedy meshing and OpenSimplexNoise, since the main reason I am embarking in this
project is to try more efficient data structures and learn something about graphics
programming.

At first (August 2022), I just did some quick and dirty experiments in Java+jMonkeyEngine, to test if the data
structures I wanted to use actually made sense. You can check it <a
href="https://git.emamaker.com/EmaMaker/voxel-test-intervalmaps/src/branch/treemaps-chunkstates-spacefilling/src/main/java/com/emamaker/voxeltest/intervaltrees/data/IntervalMap.java">here</a>.
Here's a photo, you can clearly see the greedy meshing in action.

<img alt="Here's a photo of the first experiments" src="/resources/projects/voxel2/voxel2-java.png"/> 

When I was convinced I started with C++/OpenGL (about October 2022). <a
href="https://git.emamaker.com/EmaMaker/voxel-engine">Here's the repo</a>

## Data structures

As you can read [in the notes](/notes/voxel.html), my old voxel engine divided the world into
chunks. This is pretty common to do, as not all of the world needs to be always rendered/updated.
Dividing the world into chunks allows us to dinamically load only the parts of the world we need.
This also simplifies various updating processes, again by only checking the relevant portion of the
world.

In my old engine I had a 3d-array of chunks, each with a 1d-array of blocks (the chunks are cubical,
but a 3d-array can be easily flatted into a 1d array, which is better for memory and access
times). Let's consider only chunk storage for now.

Expressing the blocks in a chunk with a 1d-array means that each index in the array represents a
block. This is fine, but it doesn't scale (memory-wise) very well. If we consider minecraft-like
worlds, we immediately notice that most of the chunk will consist of empty blocks (or air), or most
of the chunk will consist of blocks of the same type. By using an array, we will either waste memory space
(if most of the chunk is empty) or express redundant information (when most blocks in the same chunk
	are of the same type). It would be nice to have some type of lossless compression.

Octrees are another data structures often used in voxel-engines. If you know quadtrees, octrees are
their extension to 3d: we take a portion of space, and we divide it into 8 sections of equal size
(octants). Those 8 sections are then recursevely divided into 8 section and so on. You can see that
they are a tree data structure. Each node represents a portion of space, and the 8 octants are
children of this node, and are node themselves. Children of a node represent a portion of the space
of their parent.

Octrees are useful when, like in our case, the data is sparse (contains more 0s -or empty space-
	that other types of data). Sparse Voxel Octrees, as described in these two famous papers
from nvidia, are quite common, and some variations like SVODAGs (Sparse Voxel Octree
	Direct Acyclic Graph) and HashDAGs are really really interesting and clever.
They are really good if rendering happens with ray tracing (also this paper, again from nvidia) and are able to
render millions of voxels. But they really didn't click with me, they just didn't seem like the right data structure to use. I
needed something "little" in terms of chunk size, that would also be fast to probe for block
picking, meshing or light computations. After reading about them for a while, IntervalMaps really
seemed like the right choice, so I went with those.

### IntervalMap {#interval-map}

IntervalMaps also are a tree-based data structure. I first saw them mentioned by <a
href="https://0fps.net/2012/01/14/an-analysis-of-minecraft-like-engines/">0fps</a>. The idea it to
store the _"interval"_ in which a block remains the same. Let's make an example: say our chunk
stores block in a way such that they can be indexed with integers from 0 to CHUNK_SIZE^3; we could
want to have blocks from index 3 to index 8 made of type **grass**. The interval tree would store
the information not as 4 separate index in an array, but it store it as the range itself, by storing
the start and the end of the interval.

If you think about it for a moment, this is just a data structure the implements <a href="https://en.wikipedia.org/wiki/Run-length_encoding">Run-Length
encoding</a> a way that is easy to edit. Run length encoding is a compression method that stores
nearby repeating elements by storing just one element and the times it is repeated; for example the
string "aaaabbb" would be included as 4a3b (that is, 4 times 'a' and 3 times 'b').

Interval trees are nice, but we can do better. There's is no need to store both the start and the
end of each interval: it is assured that an interval ends when another one starts (as long as we
	terminate the last interval with a special terminator value). IntervalMaps are
exactly this. They are an _ordered_ tree, with each node storing a key-value pair, ordering is done
based on the keys. To increase access time performance, the tree is balanced, which assures that
access to a random node happens in O(log n) time on the number of nodes. I implemented them based on
Red-Black trees, which are `std::map` in C++ and `java.util.TreeMap` in Java. Compared to an array,
random access taking O(log n) time instead of O(1) (plus the repeated operations on different
areas of memory, instead of a contigous block) is obviously a disadvantage, but we can
try to cope by using some other tricks. Heck, no one would forbid us to <a
href="https://voxely.net/blog/the-perfect-voxel-engine/">transform the IntervalMap
into an array and operate on that, if operating on the IntervalMap directly becomes too unwieldy in
some situations</a>.

Implementing IntervalMaps themselves is not hard, however that are some corner-cases regarding
insertion that required a bit of trial and error. <a
href="https://git.emamaker.com/EmaMaker/voxel-engine/src/branch/main/include/intervalmap.hpp">This is the implementation I currently used </a>

Once IntervalMaps are working, the next point to clear is how to map 3d-space into a key for the
IntervalMap. There exist ways to map 3d space to 1d, and it can be done with the same method of
flattening 2d space to 1d.

- 2d to 1d: `index = y*maxX+x`
- 3d to 1d: `index = x+maxX*(y+z*maxY)`

The inverse is also possible, but I've yet to find a use for it. <a
href="https://stackoverflow.com/questions/20266201/3d-array-1d-flat-indexing">Here's a useful
StackOverflow post</a> on the subject. As you can see in the first answer, flattening a 3d array
into a 1d one divides the space in these _"stripes"_, in fact making it discontinuos.

A better way to map 3d space to 1d is to use a <a href="https://en.wikipedia.org/wiki/Space-filling_curve">space-filling</a> curve like <a
href="https://en.wikipedia.org/wiki/Hilbert_curve">Hilbert Curve</a>. My intuition was that a space
filling curve would better preserve local clusters of blocks, which in fact did: generating the same
world would take about 20MB less than when using Hilbert Curve compared to striping a chunk across
the X axis (contrary to my intuition, striping along the Y axis takes a couple MB than along the X
	axis), you can check [in the notes](/notes/voxel.html). I did these tests in Java, but they ported to C++ too.

Unfortunately I couldn't find any useful information on how to generated Hilbert Curves by myself,
but I found <a href="http://and-what-happened.blogspot.com/2011/08/fast-2d-and-3d-hilbert-curves-and.html">this old post</a> which provides C functions to generated
Hilbert Curves starting from Morton Order. Speaking of Morton Order, I am not the
first person who uses a space-filling curve in a Voxel Engine. In fact <a href="http://www.volumesoffun.com/implementing-morton-ordering-for-chunked-voxel-data/index.html#more-1755">this
blog posts from the authors of PolyVox</a> is what introduces me to the idea. They only used Morton Order
to index their blocks into chunks. Thinking about it, I do not recall seeing anyone else using Hilbert Curves to
index space in chunks in a voxel engine, but I think that it is more likely that I haven't searched
enough rather than noone else ever using them.

### HashMap {#hashmap}

We have a way to store blocks into chunks, but how do we store chunks?

To me, using an HashMap just seemed like the obvious choice. A key can be derived from their
coordinates (some bit-shifting, or Morton ordering again), and they can afterwards be accessed in
O(1) time. They could be also stored into a tree, ordered by their key, but that would mean at least
O(log n) access time. You will also see that this is not possible once we try to introduce
concurrency. 

## Chunk Management

In a simple update loop, the chunks around the player are inspected in concentric spheres of radius
starting from 0 up to `RENDER_DISTANCE`. If a chunk is missing, it is created. It if exists and it
has not been generated yet, we perform a generation routine that uses OpenSimplexNoise to
procedurally generate smooth terrain. If has been generated, a mesh is created and sent to the GPU
for rendering.

The algorithm is pretty simple. Remember the equation for a sphere centered at (a,b,c) of radius `r`: 

(x-a)^2 + (y-b)^2 + (z-c)^2 = r^2

And that of a circle or radius `r` centered at (a,b):

(x-a)^2 + (y-b)^2 = r^2

Where the radius is the `RENDER_DISTANCE`
    
- For every x from 0 to `RENDER_DISTANCE`
- Find the corresponding y values in a circle. By fixing x, this becomes a 2nd degrees equation,
    which yields two roots y1 and y2.
- For every y' between y1 and y2, get the corresping z1, z2 values. Now x and y are fixed and we can use the equation of the sphere to
find the values
- For every z' between z1 and z2, update the chunk at (x, y', z')

We can repeat this step every update loop, center the sphere at the player's position. The innermost
loop over z will be used to update the corresponding chunk. Again, we can use the chunk's
coordinates as an index in the [HashMap](#hash-map) using morton ordering or bitshifting.

I spent a bit of time trying to optimize this routine: finding solutions to a 2nd degree equation
requires computing a square root, and we need to do this x^3 times. This can become slow and
wasteful to repeat each update loop. An easy solution I found it to simply precompute the
coordinates centering the sphere at (0,0,0) and put them in a `std::vector`. Each update loop, instead of
calculating everything again, I simply iterate over the precalculated coordinates, translate them by
the camera position, calculate the new corresponding index and finally use it to reference the chunk
in the HashMap. This is for all intents and purposes a lookup table. Like in
[generation](#generation), and [space-filling curves](#interval-map) using lookup tables really can
cut on repeating calculations.

## Chunk Generation

My idea is to univocally assing a block type to a (x,y,z) position in the world using noise [(eventually, multiple layers
	of noise)](#future). In a way that is completely context-free. In my mind any feature in the world
should be described by a noise function (or a combination of multiple noise functions) that does not
depend on the direct knowledge of nearby blocks. This would eliminate the need to repeatedly probe
the IntervalMap for information about a single block, and the need to check blocks across multiple
chunks. 

Up until now, with the terrain consisting only of `STONE`, `DIRT` and `GRASS` this has been
rather easy: when I want to generate a chunk, I loop over all its indices (from 0 to
	CHUNK_SIZE^3), which correspond to indices in a Hilbert Curve. 
I keep a lookup table of noise values corresponding to the (x,z) coordinates of the block, which
corresponds to where, on the y-axis, the highest block in the colum at (x,z) is. This value is
basically the y coordinate of the surface level.

- Given and index, decode it into the corresponding (x,y,z) coordinates and derive the global
(x',y',z') coordinates.

- If the lookup table is empty in (x,z) column, use OpenSimplexNoise to calculate the y-height of
the column at (x', z'). Let's call this value K.

- If y' > K, the block above surface level and therefore is AIR
- If y' == K, the block is surface level and therefore is GRASS
- If 0 < y' < K the block is below surface level and therefore is STONE

- Repeat for all indices

[Here's a snippet of the code](/projects/voxel2/intro-snippets.html#generation). There are actually two different noise values. One represents surface level (where GRASS is), the
other represents how many DIRT blocks there are between the surface and STONE.

The challenge I want to tackle next is to introduce more features in the world (like
	trees or bushes) while mantaining this system context-free, but I will touch on this in a
future post. I am aware that introducing pre-generated or procedural structures cannot be done in a
context-free way, but they are outside the scope of this project (for now).

## Meshing {#meshing}

Nothing particularly special, I ported <a href="https://github.com/roboleary/GreedyMesh/blob/master/src/mygame/Main.java">roboleary's port of</a> <a href="https://github.com/mikolalysenko/mikolalysenko.github.com/blob/gh-pages/MinecraftMeshes/js/greedy.js">0fps'
algorithm</a> to C++. Put basically, instead of creating a quad for each exposed face, greedy
meshing stitches adjacent quads into a bigger one (making sure of repeating -and not stretching- the
	texture!). <a href="https://0fps.net/2012/06/30/meshing-in-a-minecraft-game/">0fps has a good article explaining it</a>.

My old engine also used Greedy Meshing, and in a way that didn't differ much from 0fps', but I like
his way of doing things more. Plus I wanted something do get up and running fast, to experiment with
other stuff in the engine. So I ported this code, with the promise to myself of trying different meshing
methods in the future (maybe <a
	href="https://git.emamaker.com/EmaMaker/raymarching">raymarching</a>?).

We can also enhance greedy meshing with face-culling, to not render hidden faces of adjacent solid blocks.
Those blocks are the vast majority of the blocks present in our world, so our GPU will thank us,
      since it would be otherwise rendering stuff we will never be able to see. Face culling is
      something that roboleary's code did not do, and I added in.

This kind of face culling is different from OpenGL's face culling. The latter renders a face only if
seen from the proper side, as described by the order (clockwise or counter clockwise) of the indices
of its vertices. It doesn't hurt, so <a
href="https://git.emamaker.com/EmaMaker/voxel-engine/src/commit/73f38e5d2f1767932127301707d1e430d8847b56/src/main.cpp#L59">I
enabled that too</a> and made sure I got my indices right in meshing.

<img class="img-side-by-side" src="/resources/projects/voxel2/meshing-fill.png" /> 
<img class="img-side-by-side" src="/resources/projects/voxel2/meshing-greedy.png" /> 

Currently, for every vertex, this data is sent to the GPU:

- 3 floats for position in space
- 3 floats for normal
- 3 floats for texture coordinates

That means, for a single quad to be put on screen a total of 9\*4=36 floats are sent to the GPU,
     totaling 36\*4=144 bytes.

Why _3_ floats for texture coords? Instead of using a texture atlas, I am using 3D-texture arrays.
This is basically a texture that instead of having different mipmap levels has effectively different
textures on the third dimension. This makes it easier to use block ids to reference the texture and
doesn't require additional computation in the fragment shader to support repetition: we can just set
`GL_TEXTURE_WRAP_S` and `GL_TEXTURE_WRAP_T` to `GL_REPEAT`.

I also set `GL_TEXTURE_MIN_FILTER` and `GL_TEXTURE_MAG_FILTER` to `GL_NEAREST` to give the textures
a pixel-y look which I really enjoy.

Normals are required for phong-lighting to work, so I cannot remove them. But since we now that the
`CHUNK_SIZE` will never be greater than 255 (`CHUNK_SIZE` is just a variable in the code, and after
	a bit of experimenting I set that to 32) we can ditch floats to express texture coordinates
and use bytes instead. This brings the data for each vertex to:

- 3 floats for position in space
- 3 floats for normal
- 3 bytes for texture coordinates

Which means, for a single quad, 6\*4=24 floats and a total of 24\*4+3=99 bytes.

This did have the tangible effect of cutting the VRAM usage by about 10MB in the test world
(`CHUNK_SIZE`=32, `RENDER_DISTANCE`=16, noise seed 12345)

_I think_ I could further shave this down by using bytes for everything. This should be possibile
since a vertex' position and normal only ever assumes integer values: a block is a 1-unit cube, and
	a normal only ever has a single axis set to 1 or -1, the others to 0. Furthermore, vertex positions are
	relative to the chunk they belong to. During a _draw_ call, the **chunk's model matrix**
	handles the translation in the world. so they cannot exceed `CHUNK_SIZE`, which I keep at 32

I could also further shave down the amount of data sent to the gpu by sending partial information about a single
vertex (and the quad it is part of) to the GPU, then let a **geometry shader** handle the rest.

Another thing I will try in the [future](#future), I guess.

<img alt="A landscape" src="/resources/projects/voxel2/landscape-wireframe.png"/> 

## Concurrency

This is where my engine and my Nintendo Wii Part ways.

Up until now, I've talked about the engine as if everything happened in a single thread. Because, in
fact, it did: everything happens in the main thread. The problem with this is that operations like
chunk generation or chunk meshing take sometimes up to a couple milliseconds (in the worst case, a
millisecond in the average case). Generation is a lot better than meshing, but still those are the
two most time-consuming operations in the engine. If aiming for a stable 60FPS, a single frame will take 16.6ms. Spending a whole millisecond meshing the chunk means that at most we can mesh 16 chunks
per frame, which even at 60FPS does not account for the hundreds of chunks that might need to be
generated when the camera moves outside the already generated region. As it is, generating new chunks stales the whole engine for about a
second, because meshing and generation are done in the same thread as rendering and input listening.

We could try to split jobs into time batches so that, aiming for 60FPS, we try to do as much work as possibile in a frame,
     without exceeding the 16.6ms mark, and the carry the remaining work to the next frame.
But, already at a `RENDER_DISTANCE` of 16, this makes the work carry on for quite some time.


The other option is to open a new thread which takes care of generating and meshing new chunks,
    which in turn introduces the problem of concurrent access to data.

Initially I created two new threads, one for generation and one for meshing. The main thread would
iterate over chunks, checking which chunks needed which jobs, then adding them to respective queues for each
thread. I used `std::mutex`es to avoid different threads writing to or reading from the same queue at the same time. This
actually works fine, and removes the spikes during generation/meshing. However, it turns out that
continously trying to lock/unlock or check on mutexes is kinda slow, so doing it hundreds of time in
a second leaves a lot of performance on the table.

### Concurrent Data Structures

The main problem here is the need to avoid data races on the queues used to exchange information
between threads. The reason is that data structures in the standard library do not support
concurrent access, so mutexes are a must. The possible solution is to use other data structures that
do support concurrent access. The main choice was between the <a href="https://www.boost.org">boost library</a> and the
<a href=ahttps://github.com/oneapi-src/oneTBB"">oneAPI TBB (Threading Building Blocks)</a> library. I ultimately decided to use oneTBB
because it offers more concurrent data structures, mainly chosed oneTBB because it offered `concurrent_hash_map`s, which are an easy replacement for
the HashMap I'm already using. Also `concurrent_hash_map` is the only container in all of TBB that
supports **safe concurrent deletition** of an element.


Moving to oneTBB was not an easy task: I had to wrap my head around new concepts that, honestly, I
don't feel I've grasped yet, and there are still some bugs to iron out, mostly due to freeing of
memory. The job certainly wasn't made easier by the poor documentation
available. The website linked on GitHub is, in general, lacking and I don't even remember how I
managed to get to the <a
href="https://spec.oneapi.io/versions/latest/elements/oneTBB/source/containers/concurrent_hash_map_cls.html">full
specification documentation</a>. By reading on stackoverflow and intel's oneAPI forum, I don't look
like the only person having problems with oneAPI. I also found a book, titled "Pro TBB - C++
Parallel Programming with Reading Builiding Blocks", but it really
doesn't go much further that the specification documentation.

Rants aside, I'm now at the point where:

- Render + Input happen in the main thread
- Chunk carry an atomic bitfield (currently a `uint8_t`) called `CHUNK_STATE`, which holds information on:
    - If chunk has been generated
    - If chunk has been meshed
    - If chunk mesh has been loaded to VRAM
    - If chunk is out of RENDER_DISTANCE
    - If chunk needs to be freed from memory (if out of render distance for more than a fixed time)

- Chunk updating, meshing and generating happen in a second thread
- Only a set number of chunks are meshed at the same time, once a mesh is sent to the gpu it "frees
a spot" for another chunk to be meshed. 

### Mesh Data exchange

The last point avoids binding `std::vectors` for mesh data (vertices, indices, uv coords) to each chunk,
    and continously filling and freeing said memory. Exchange between the update/meshing thread and
    the main thread happen via a struct called `MeshData`, containing the following info:

```
struct MeshData{
    Chunk::Chunk* chunk;
    GLuint numVertices{0};

    std::vector<GLfloat> vertices;
    std::vector<GLfloat> colors;
    std::vector<GLuint> indices;
};
```

The "freeing a spot" mechanism is done with the concurrent queues. Initially the mesher thread has a
queue full of a set number X of MeshData structures. The main thread also has a queue, but empty. 
During greedy meshing, one of these structs
gets popped from the mesher thread's queue and filled with the appropriate data. At the end of
greedy meshing, the struct full of data is pushed into the main thread's queue. The main thread in
turn, at each _render_ call, pops a MeshData struct from its own queue, sends the data to the GPU,
    then clears the struct's fields and pushes it back into the mesher thread's queue.

This process is needed because each OpenGL context is bound to a single thread and a single window.
Trying to send data from the mesher thread directly to the GPU would require another OpenGL context
separate from the main one, which in turn would mean another window and that is pretty useless for
the engine.

### IntervalMap concurrency

Now, each chunk's IntervalMap is not concurrent. To avoid problems, the `CHUNK_STATE` bitfield is also
used to manage access to the IntervalMap. Remember `CHUNK_STATE` is atomic, so it doesn't pose a
problem for concurrent access. 

For example, when a chunk is being generated, the
`CHUNK_STATE_GENERATED` bit is set to false, and if it is false, trying to call `getBlock()` on
that chunk, will return `Block::NULLBLK`, which is a value that I use for error signaling.

This is nice, and the performance is way better (I almost doubled the FPS), but we can do better.
For example, when placing or breaking a new block, the chunk's mesh has to be regenerated. If there
are a lot of meshes to regenerate, some time can pass between clicking the mouse and the chunk
actually changing. In the future, I'd like to move (again) meshing and generation in their own
threads, and using concurrent priority queues to pass information to those threads. In this case, a
chunk which had a block removed should have a higher priority in meshing that a chunk part of normal
world generation.

Using a concurrent HashMap to store chunks, also means that, while meshing a chunk, neighbouring
chunks can also be probed to check for blocks and not render useless borders to each chunk, which
means less quads to draw and a happier GPU.

# Conclusion

It has been a blast! I'm really enjoying building this engine and there is a lot of stuff I want to
try. I probably should've started writing about it sooner, now I have a lot of backlog :)

The wall of text you just read condenses down about 6 months of on-and-off work, composed mostly of
blocks of 1 to 2 hours that I could (rarely) cut for myself during the week after university (and a
	iatus of about a month and a half where I worked on a little <a
	href="https://git.emamaker.com/EmaMaker/raymarching">raymarching engine</a>). 

Still, I discovered that I really like
working at this pace: I can spend time during the week (mostly during commutes, on lunch breaks at
	uni, or in general some free time that is not enough to work on anything concrete) 
by learning useful information to put into this engine and thinking about what I wanted
to do and how I wanted to do it, so when I had a bit of spare time I knew exactly what to do and how
to do it, avoiding, well _sparing_, time.

You can see the progression of this project via git:
<a href="https://git.emamaker.com/emamaker/voxel-engine">Project Repo</a>
<a href="https://github.com/emamaker/voxel-engine">Mirror</a>


Below is also a little video wandering around in the world.

## For the Future {#future}

There are lots of things I still want to try and write about, here are some:

- Better procedural generation with multiple octaves of noise
- Trees, bushes and similar features in the world
- Shaving down information sent from CPU to GPU
- Use of geometry shaders to complete mesh generation on GPU and reduce data sent from CPU to GPU
- Flood lighting, like in <a href="https://web.archive.org/web/20210622035752/https://www.seedofandromeda.com/blogs/29-fast-flood-fill-lighting-in-a-blocky-voxel-game-pt-1">Sea of Andromeda</a>
- Shadows
- Ambient occlusion (baked?)
- Try different space-filling curves, like the <a
href="https://en.wikipedia.org/wiki/Peano_curve">Peano curve</a>, which seems like could better
preserve local clusters of blocks.
- A UI with dearImGui


# Resources {#resources}

Links to resources I used in this project. Should they become unavailable, I have a copy locally
on my computer, you can email me if you need.
