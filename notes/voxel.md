Title: Voxel notes

# Old voxel engine:

## Data structure
Used 1d arrays for storing blocks in Chunks. This is good enough, 1d arrays are faster to poll than 3d arrays. Memory usage shouldn't be that bad.

Used 3d arrays for storing chunks in the world, this is VERY VERY bad, there's a lot of wasted memory. Each update loop the engine looked in a sphere around the player of radius RENDER DISTANCE and updated each chunk that needed it. Then each chunk, using a JME3 Abstract Control was updated each loop and the chunk was removed or added to the scene if needed. This infact created a constant usage of memory, but a limited world. 

## Save files
Saved and loaded from file by saving every single block, again very very bad. Could create up to GIGABYTES in save files

# New voxel engine

## Data Structure
Using interval trees to store blocks in Chunks, as suggested by [0fps](https://0fps.net/2012/01/14/an-analysis-of-minecraft-like-engines/) and on [reddit](https://teddit.pussthecat.org/r/VoxelGameDev/comments/elap1u/using_interval_maps_instead_of_arrays_for_storing/). Thought about using SVOs (Sparse Voxel Octrees) as described in the nvidia paper or even SVDAG (Sparse Voxel Direct Acyclic Graph) but they just don't look like the right tool for job.

Chunks are then stored into an hashmap as they get created and destroyed. This ensured O(1) access to a Chunk and good memory usage, while not giving limits to the size of the world
Interval trees kinda give built-in memory compression like in Run length encoding, with O(log n) average and O(n) worst case time to add or remove nodes. I'm not really removing nodes right now when needed, and they tend to grow pretty large in memory footprint as of (15/08/2002).

Actually Interval Trees don't really seem like the best tool for the job either, as they require a start and an end to an interval, while I just really need to know the start of the interval (like in Run Length encoding) representing a run of identical blocks, and assume that it will either continue until the end of there will be a new node in the tree representing the start of the new interval right after it. I will probably be better off using some kind of Binary Research Tree, storing a key-value pair where the key is the start of the run (interval) and the value is the type of block. Probably the best option is to use a Red-Black Tree instead of a plain Binary Research Tree. This is similar to what the [reddit guy](!missing_link) did, except he was using C++ IntervalMaps, which are implemented as Red-Black Trees 

### Update
I implemented IntervalMaps starting from Java's java.util.TreeMaps, which are implemented as Red-Black Trees, and had a huge reduction in memory usage
improvement (down to 80-85 MB of memory from 400+).
For further compression, i think some kind of space-filling curve could compress the tree even further by better expressing local conglomerates of the same kind of blocks in the 3d-to-1d flatteing. By researching space-filling curves, on [wikipedia](!https://en.wikipedia.org/wiki/Space-filling_curve), I'm lead to think that an Hilbert Curve is better than [Morton](!https://en.wikipedia.org/wiki/Z-order_curve) curves or Morton ordering for my scope, [although I've seen morton ordering used in big voxel engines projects](!http://www.volumesoffun.com/implementing-morton-ordering-for-chunked-voxel-data/index.html). I will try both and check which one compresses better.
By researching 3D Hilbert curves, i stumbled upon [this website](https://eisenwave.github.io/voxel-compression-docs/rle/hilbert_curves.html), which brings to [this other site for C implementations](!http://and-what-happened.blogspot.com/2011/08/fast-2d-and-3d-hilbert-curves-and.html)

### Update 2
Implemented Hilbert Curves and re-run benchmarks on all branches and techniques used until now. I apparently had huge problems in my old benchmarks, because I reported very different numbers across all tests, even on stuff I haven't touched in a while.

I tested these three techniques:
- Interval Trees, with my own (crappy in some ways) implementation, with nested iteration as a 3d-to-1d flattening
- Interval Maps, with nested iteration as a 3d-to-1d flattening
- Interval Maps, with hilbert curves

They all use chunk states instead of queues, as continously filling and empting queues was not making the garbage collector happy and some huge memory usage spikes could be seen in VisualVM

In the end, all the techniques seem to perform about equal in my base benchmark (creating a cube of render_distance*2 side, generated with arrayGenerateCorner. Cubical chunks with a side of 16 voxels), reporting:

RENDER_DISTANCE = 8

- Chunk size 32 (just creating chunks, no world update)


	| Type	| Normal Heap (used/allocated) |	After 1GC (used/allocated)  |
	|-------|------------------------------|--------------------------------|
	| Interval Trees (nested iteration) | 74/154 | 32/112 | x
	| Interval Maps (space filling)  | 128/210 | 26/98 | x


- Chunk size 16 (normal world update loop)


	| Type	| Normal Heap (used/allocated) |	After 1GC (used/allocated)  |
	|-------|------------------------------|--------------------------------| 
	| Interval Trees (nested iteration) | 148/196 | 81/196 | x
	| Interval Maps (space filling - Hilbert Bits 2)  | 100/193 | 59/193 | x
	| Interval Maps (space filling - Hilbert Bits 4)  | 88/192 | 55/193 | x

To be noted:
- IntervalTrees seem to have a less "spikey" use of ram, but that could be due to other causes rather than the data structure itself
- Using a spacefilling curve requires more calculations than nested iteration, but that can be optimized
- The manual GC call just cuts the allocated heap, as the used heap already falls on its own once chunk generation has finished

I honestly cannot explain myself how intervaltrees can have such a similar memory usage to intervalmaps. On one hand, I know that my implementation of intervaltrees lacks the removal of adjacent similar nodes, and that leaves a lot of unneeded nodes around. Maybe this could be better noted once block picking is added. As of right now, this situation doesn't really occur.
On the other hand, java's implementation of TreeMaps has some stuff that I don't really need, and the memory size of the structure itself could be reduced if I were to reimplement it to better suit my needs.

Also I haven't tested interval trees with a space filling curve, but I don't think that would change much.

## Ideas
 - I'm currently using greedy meshing to obtain a triangle mesh out of the interval tree, by first converting it into an array. It is possible that, by changing the function i use to flatten a 3d index into a 2d one, the interval tree could be used directly into the greedy meshing function without a preliminary conversion to an array. The use of an array could still be need for fast chunk generation though
 - If going with bigger chunks, probably it will be best to just transform the tree into an array past a certain height (meaning very different blocks)
 - At some point, chunks need to be unloaded from memory. This can be done in three stages:
	1) Chunks within render distance are shown and eventual logic gets updated
	2) Chunks little outside of render distance are show but no logic gets update (what about LODs?)
	3) Chunks that havebeen far from render distance for a short time are kept in a separate hashmap with just the deltas stored. After enough time has passed, they get dumped to the HDD and completely unloaded from memory

	This method makes an assumption on chunk structure, such that generation is done at runtime and only modifications are permanently stored. This also means that generation data and modification data are stored in two different places, possibly using two different trees, a tree and an array, or something else
 - SVDAGs look like an interesting structure to look more deeply into and try to implement in some way. Maybe textures will be difficult to implement, but they would probably give access to some non-cubical blocks, by making them up of blocks smaller that the 1-Unit size decided for a single voxel
