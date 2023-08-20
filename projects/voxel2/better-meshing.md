title: Better Mesh Generation for my Voxel Engine

# Better Mesh Generation for my Voxel Engine

As I explained in my <a href="/projects/voxel2/intro.md">first post</a> about my voxel engine,
   a mesh for a chunk is generated using the greedy meshing algorithm, inspired by <a
   href="https://0fps.net/2012/06/30/meshing-in-a-minecraft-game/">this famous blog post</a> by
   0fps. <a href="https://github.com/roboleary/GreedyMesh">roboleary</a> ported 0fps' javascript algorithm to Java, and I ported that
   to C++.

   Actually in my <a href="/projects/voxelengine.md">old engine</a> I was already using greedy
   meshing to generate the mesh for every chunk. The implementation was similar, at least in the
   idea, except that I had 3 different switch-cases to march along the correct direction for
   every face. I just find 0fps/roboleary's implementation much more elegant than mine. I know I
   could fix it, but still, when I started working on the engine I just wanted to get this part
   running quickly, so that I could experiment with other stuff like multithreading,
   this article, and nicer world generation (more interesting terrain features, decorations -
	   which are the argument of an upcoming article, biomes...)


## What's wrong with Greedy Meshing?

Well nothing, almost. Not in the algorithm itself at least. The problem is how I move data from
the CPU, which executes the meshing algorithm, to the GPU, which puts the pixels on the screen.

Currently, for every vertex of the mesh I send:

- 3 floats for position
- 3 floats for normals
- 3 floats for texture

So a float is 4 bytes, which adds up to 3\*3\*4=36 bytes **for each VERTEX**. That's an awful
lot of data for a single vertex.

Now, in my quest to learn OpenGL I read pretty early about <a href="https://www.khronos.org/opengl/wiki/Geometry_Shader">Geometry Shaders</a>.
Those are pretty cool: they execute after the vertex shader, take as input a set of
vertices and output multiple vertices, manipulated as we wish, packed into a <a
href="https://www.khronos.org/opengl/wiki/Primitive">primitive</a>.

I thought I can use a geometry shader to optimize my greedy meshing routine. Since I'm
making a cubical voxel engine, ultimately we 
want the meshing stage to output the quads that make up the faces of the cubes (most probably
	multiple cubes merged together). But generating
whole quads on the CPU is really a waste of resources, especially memory, and is also wasting
bandwidth between the CPU and the GPU.

Now, we can generate a single point on the CPU, representing the whole quad that would be
generated instead, which carries the following information:

- 3 floats for bottom-left position
- 3 floats for _extents_. In other words, the size of the quad.
- 1 float for the block type
- 1 boolean for _back face_. This represents which side on the same axis the quad is on
(right/left, top/bottom, from/back)

I call the set of these a _cloud of points_.

Ultimately it's almost the same amount of data _per element of the array_, but I'm sending 4
times less data, since an element of the _cloud_ represents a whole quad, while previously an
element of the arrays that were sent to the GPU represented a single vertex (of which four are
	needed to make a quad).

All the vertices of the quad have the same normal, which can be calculated from the _extents_
vector. One and only one element of this vector is zero: this represents which axis the quad
is on. Combined with the _backface_ element, it can be used to determine the normal of the
vertex. This is done in the vertex shader, and sent to the geometry shader. The _extent_ vector
is also used to determine the texture coordinate of each vertex during the geometry
shader.


## The Geometry Shader itself

It's up to the geometry shader generate the new vertices. It also comes with a little added
benefit: if we get the order in which to output the vertices right, we can output a
triangle_strip primitive. A triangle strip is just like a triangle, except it doesn't need a
separate indices array to order the vertices: in a triangle strip any 3 consecutive vertices make a triangle. This
could also be implemented on the CPU, but I only discovered them when working on this :) .

After the cloud of points is sent to the GS, together with the calculated normal for each
vertex, the GS procedes to generate new vertices in the order shown below. It's this order that
guarantees that a triangle_strip is generated.

In a GS, a new vertex is created by calling `EmitVertex()`. After all vertices have been
created, the primitive is closed my calling `EndPrimitive()`. Any data about the vertex
(including data that needs to be sent from
the geometry shader to the fragment shader) must be set before calling `EmitVertex()`

Let's breakdown the GS:

1. Setup inputs and outputs.

    ```
    layout (points) in;
    layout (triangle_strip, max_vertices = 4) out;

    in VS_OUT{
	vec3 Extents;
	vec3 Normal;
	float BlockType;
    } gs_in[];

    out vec3 TexCoord;
    out vec3 Normal;
    out vec3 FragPos;

    uniform mat4 view;
    uniform mat4 projection;

    ```

Most of the variables are self-explaining. TexCoord is the texture coordinates (<a
	href="/projects/voxel2/intro.html">remember, they are 3d</a>) and FragPos is the position
of the fragment in world coordinates, used for lighting calculations in the fragment
shader.

Also note that each vertex needs to be translated in its position as
seen by the camera, so it needs to be multiplied by the view and projection matrices. This
was previously done in the vertex shader, but now needs to be done for each vertex before
emitting it.

2. Entering the `main()` we start emitting vertices

    ```
    Normal = gs_in[0].Normal;

    TexCoord = vec3(0.0, 0.0, gs_in[0].BlockType);
    gl_Position = gl_in[0].gl_Position;
    FragPos = vec3(gl_Position);
    gl_Position = projection * view * gl_Position;
    EmitVertex();
    ```

The normal is the same for every vertex, so it only needs to be set once at the start. The
first vertex is bottom-left, which corresponds to the position vector (`gl_in[0].gl_Position`).
Vertices between the vertex and the geometry shader are always passed as arrays, even if there's
just one vertex like in our case

3. Now we have to procede based on which axes the quad is parallel to: the quad is parallel to
the axes whose extent is not 0. This can be checked with an
if statement.

    ```
    if(gs_in[0].Extents.x == 0){
	...
    }else if(gs_in.Extents.y == 0) {
	...
    }else{
	...
    }
    ```

    The bodies of the statements are pretty similar, only the exact values change. Let's examine a quad parallel to the y and z axes

    4. Emit second (bottom-right) vertex

    ```
    TexCoord = vec3(0.0, gs_in[0].Extents.z, gs_in[0].BlockType);
    gl_Position = gl_in[0].gl_Position + vec4(0.0, 0.0, gs_in[0].Extents.z, 0.0);
    FragPos = vec3(gl_Position);
    gl_Position = projection * view * gl_Position;
    EmitVertex();
    ```

Again, pretty similar to the first one.

5. And so is the third (top-left) one

    ```
    TexCoord = vec3(gs_in[0].Extents.y, 0.0,  gs_in[0].BlockType);
    gl_Position = gl_in[0].gl_Position + vec4(0.0, gs_in[0].Extents.y, 0.0, 0.0);
    FragPos = vec3(gl_Position);
    gl_Position = projection * view * gl_Position;
    EmitVertex();
    ```

6. After the third vertex, and still in the if statement body, we set the texture coordinate for
the fourth vertex

    ```
    TexCoord = vec3(gs_in[0].Extents.y, gs_in[0].Extents.z,  gs_in[0].BlockType);
    ```

7. The fourth and last vertex (top-right) is independent of axes, because it's always at
_position_+_extent_. Its TexCoord is not however, and has been set before in the if statement
body.

    ```   
    gl_Position = gl_in[0].gl_Position + vec4(gs_in[0].Extents, 0.0);
    FragPos = vec3(gl_Position);
    gl_Position = projection * view * gl_Position;
    EmitVertex();
    ```

8. And finally

        EndPrimitive();

This order of emitting vertices guarantees a working triangle_strip. However, it has the downside
of not respecting the vertex order for face culling anymore, so I disabled it for the moment. It
could be re-enabled by using another switch case to check the normal of the vertices and
eventually emitting them in reverse order.

## Why floats?

Using integers would save up a lot more memory on both the CPU and the GPU. Since a chunk is set
to be 32 blocks wide, neither the _position_ nor the _extents_ vector can ever go past the value
31: a byte will be more that sufficient to express it, instead of the 4 bytes always required
by a float. With some bitwise operations, it
could be compressed even further.

Too bad that GPUs **SUCK** at integer calculations!

Nah, that's not actually true, at least not for modern GPUs.

I've never posted on about my
desktop computer, but the gist of it is that I built it during the high of the GPU
shortage of 2020 and got no GPU for it. So I bought a used AMD Radeon HD6850, which is now more a
than 10 years old GPU, with the intent to have a dual GPU setup in the future and pass one GPU to a Windows 10
machine to do CAD work. Some time later I bought a NVIDIA Tesla M40. This is a server GPU, basically
a Titan X Maxwell from 2015, except with a cooler that makes absolutely no sense for a
desktop computer (which makes keeping it from overheating a challenge) and no video output. So I kept the HD6850 to have a video output, while
offloading the heavy calculations to the Tesla M40 with <a
href="https://wiki.archlinux.org/title/PRIME">PRIME</a>.

Where was I? Oh right, GPUs suck at integer calculations, or at least old ones do. My Radeon HD
6850 is indeed an old GPU: it was released in 2010. My Tesla is somewhat newer -still a bit
old- but was an enterprise-grade GPU when it got out. And due to the nature of my setup, I can test the engine on both GPUs.

The morale of the story is that I did try using (unsigned) integers instead of floats. While the M40 didn't
break a sweat, the HD6850 was literally chocking. Frametimes more than decupled, with framerate
dipping
below 15 FPS as soon as more than a handful of chunks had to be rendered.

Since I aim to keep
compatibility with older hardware as much as possible (this engine runs at about 80FPS at 8
	chunks render distance on my ancient Dell M1330, released in a 2008 with a whopping
	Intel Core 2 Duo T7000 and a NVIDIA Gefore 7800GS) I decided against the
transition to integers to express mesh data, even if it meant giving up a big decrease in RAM
and VRAM usage.

# Conclusions

The reason I decided to introduce a Geometry Shader was to see if I could reduce the VRAM usage,
    and if so, by how much. Reducing VRAM usage means that I can make the world generation more
    interesting (i.e. more detailed terrain, decorations like trees) without wasting
    resources.

Here's a table of the VRAM usage with after the introduction of the Geometry Shader in
meshing. Tests are done in my standard world, you can <a
href="/notes/voxel-vram.html">check the notes</a>.

|Type|VRAM usage (MB)|
|----|----------|
|No Geometry Shader (old) | 82 |
|With Geometry Shader | 20 |
|With Geometry Shader + integers instead of floats | 8 |


Reducing the amount of data that has to be sent from the CPU to the GPU has also the collateral
effect of reducing RAM usage on the CPU, which was only partially a problem because mesh
data was cleared as soon as it was sent to the GPU, but it still required a lot of operations
to grow the heap to fit the data, only for it to be flushed afterwards.


Source code is available on <a
href="https://git.emamaker.com/emamaker/voxel-engine">Gitea</a> <a
href="https://github.com/emamaker/voxel-engine">(GitHub mirror)</a> <a
href="https://codeberg.org/emamaker/voxel-engine">(Codeberg mirror)</a>
