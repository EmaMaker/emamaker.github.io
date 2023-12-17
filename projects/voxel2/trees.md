title: Generating Trees | Voxel Engine #3

# Generating Trees | Voxel Engine #3

The world looks quite bland at the moment, let's add some trees!

First, let's recap how world generation currently works, then we'll see how to add trees in a way
    that makes sense.

## Current world gen

As I wrote in the [intro](/projects/voxel2/intro.html), I'm trying to keep the chunk generation
algorithm context-free for as much as I can. I already know this won't work if I ever want to add
more complex structures, but I'm not planning for it anytime soon. But if I ever wanna do that, I
might switch to [something layer based](add links).

Also recall that blocks in a chunk are stored in an interval map, which is a tree-based data
structure that performs lossless compression, basically run-length encoding. The current chunk
generation algorithm takes advantage of the data structure, by iterating through the chunk in the same
order as blocks will be indexed in the interval map, to take advantage of its
compression properties. Specifically, the blocks are indexed to form a
a Hilbert curve, which is a particular type of space-filling curve. This means that the chunks are
not necessarly (actually, more often than not) iterated following a particular cartesian axis.
A noise value, calculated using Open Simplex Noise is calculated at every `(x,z)` coordinate. As an
optimization, noise values are calculated only once per coordinate and help in lookup tables:

- `grassNoiseLUT` stores the surface level.
- `dirtNoiseLUT` stores how many DIRT blocks there are below surface level
- Everything between 0 and `grassNoiseLUT(x,z)-dirtNoiseLUT(x,z)` is set to STONE
- Everything above `grassNoiseLUT` is set to AIR

Now, on top of this system, we need to add tree generation. At first it might seem easy: just add
another lookup table, vary the tree trunk height using noise and call it a day. This is correct,
	however it poses some problems, first of which is that trees without leaves are kinda sad.

## Tree trunks

While the solution I just proposed works, it just _kinda_ works. Let's put leaves aside for now and
concentrate on tree trunks first.

The first problem to arise is that using OpenSimplexNoise does not allow us to decide _where_ to place a trunk either, we can just sample
noise at a point and place a trunk if it is over a given threshold. And OpenSimplexNoise is
a _slow_. While we could ignore slowness, the fact that we cannot decide where to place a trunk
means that we have no way to avoid generating two adjacent trunks. If
    you squint your eyes just a little bit, you can notice a lot in this screenshot.

<img alt="Lots of adjacent tree trunks" src="/resources/projects/voxel2/trees-adjacent.png"/> 

I couldn't find a solution to this problem will keeping to use noise. Probably there is one, but I
guess finding it is just not worth the hassle.

What I decided to do is instead section up the world in _cells_. Those are different from chunks, in
that they divide the world just in 2d (x,z) and not in 3d (x,y,z). Also they have different dimensions
(but are still square),
     such that a single chunk may be spanned by multiple cells (or viceversa).
     We impose that at most one tree may be generated _per cell_, maybe
     with some offset to avoid having them appearing like a grid of points. Note that when adding an
     offset from the center, the maximum offset must be strictly less than half of the side of the cell, again to
     avoid having two adjacent trunks.

At this point, for every `(x,z)` point we can check which tree cell it falls into and compute the `x`
and `z` chunk coordinates at which a tree trunk shall be placed for said cell. Offsets and trunk height can be
generated pseudo-randomly or using noise, as long as the function stays deterministic (that is to
	say, at same inputs should always correspond the same output). Here's how I do it:

```
#define LEAVES_RADIUS 3
#define WOOD_CELL_SIZE 13
#define WOOD_CELL_CENTER 7
#define TREE_STANDARD_HEIGHT 7
#define TREE_HEIGHT_VARIATION 2
#define WOOD_CELL_BORDER (LEAVES_RADIUS-1)
#define WOOD_MAX_OFFSET (WOOD_CELL_SIZE-WOOD_CELL_CENTER-WOOD_CELL_BORDER)

const int TREE_MASTER_SEED_X = mt();
const int TREE_MASTER_SEED_Z = mt();
struct TreeCellInfo evaluateTreeCell(int wcx, int wcz){
	int anglex = TREE_MASTER_SEED_X*wcx+TREE_MASTER_SEED_Z*wcz;
	int anglez = TREE_MASTER_SEED_Z*wcz+TREE_MASTER_SEED_X*wcx;

	// Start at the center of the cell, with a bit of random offset
	int wcx_off = WOOD_CELL_CENTER + WOOD_MAX_OFFSET * sines[anglex % 360];
	int wcz_off = WOOD_CELL_CENTER + WOOD_MAX_OFFSET * cosines[anglez % 360];

	struct TreeCellInfo result{};

	// Cell to world coordinates
	result.trunk_x = wcx * WOOD_CELL_SIZE + wcx_off;
	result.trunk_z = wcz * WOOD_CELL_SIZE + wcz_off;

	result.trunk_x_offset = wcx_off;
	result.trunk_z_offset = wcz_off;

	return result;
}

```

Offsets are determined using a simple function of the coordinates and some random seeds, then taking
the sine and cosine. The idea is that the numbers will be random enough (from the seeds) at that
they will at least be out of phase with each each other (by taking the sine and the cosine
respectevely) such that trunks won't always be generated along the diagonals of a cell.

All these results are held in a dedicated struct:

```
// Info on the tree cell to generate
struct TreeCellInfo{
    // Cell coordinates (in "tree cell space")
    int wcx, wcz;
    // trunk offset from 0,0 in the cell
    int trunk_x_offset, trunk_z_offset;
    // Global x,z position of the trunk
    int trunk_x, trunk_z;
};
```

and then saved into a lookup table, which is evaluated at the start of generation, for each chunk.
This is done as an optimization, and speeds things up sensibly (I haven't taken any measurement, but
	the difference could be easily felt by eye even on my Ryzen7 3700X/RTX 3060 desktop).

```
std::array<TreeCellInfo, TREE_LUT_SIZE*TREE_LUT_SIZE> treeLUT;

// Tree LUT
int tree_lut_x_offset = cx / WOOD_CELL_SIZE - 1;
int tree_lut_z_offset = cz / WOOD_CELL_SIZE - 1;

for(int i = 0; i < TREE_LUT_SIZE; i++)
    for(int k = 0; k < TREE_LUT_SIZE; k++){
	int wcx = (tree_lut_x_offset + i);
	int wcz = (tree_lut_z_offset + k);
	treeLUT[i * TREE_LUT_SIZE + k] = evaluateTreeCell(wcx, wcz);
    }

```

Now when iterating on a block we look up the corresponding cell in the table. If the cell says that
the current block coordinates are also the coordinates of its tree trunk (and `y >= surface_level &&
	y <= trunk_height`) we set the current block as WOOD.

Now you can see that there are no adjacent trees


<img alt="Lots of adjacent tree trunks" src="/resources/projects/voxel2/trees-nonadjacent.png"/> 

## Leaves

Those are tricky! But luckily, we already did most of the heavy lifting computing trunks.

The problem with leaves is that _somehow_, when trying to place a leaf block, we need to reference
the position and the height of the trunk it is attached to. It's unfeasable to let every single leaf
block being placed check a radius of `LEAVES_RADIUS` and look for a trunk which may or may not be
its origin.

Luckily we have a lookup table which we can reference. When iterating a block, we check which tree
cell it corresponds to. By referencing the `TreeCell` lookup table, we know where the trunk is in
global coordinates and how tall it is, so that we can start placing leaves from the uppermost block
of the trunk. We can add this information into the struct.

```
// Info on the tree cell to generate
struct TreeCellInfo{
    // Cell coordinates (in "tree cell space")
    int wcx, wcz;
    // trunk offset from 0,0 in the cell
    int trunk_x_offset, trunk_z_offset;
    // Global x,z position of the trunk
    int trunk_x, trunk_z;
    // Y of the center of the leaves sphere
    int leaves_y_pos;
};

// Tree cell Info
const int TREE_MASTER_SEED_X = mt();
const int TREE_MASTER_SEED_Z = mt();
struct TreeCellInfo evaluateTreeCell(int wcx, int wcz){
	int anglex = TREE_MASTER_SEED_X*wcx+TREE_MASTER_SEED_Z*wcz;
	int anglez = TREE_MASTER_SEED_Z*wcz+TREE_MASTER_SEED_X*wcx;

	// Start at the center of the cell, with a bit of random offset
	int wcx_off = WOOD_CELL_CENTER + WOOD_MAX_OFFSET * sines[anglex % 360];
	int wcz_off = WOOD_CELL_CENTER + WOOD_MAX_OFFSET * cosines[anglez % 360];

	struct TreeCellInfo result{};

	// Cell to world coordinates
	result.trunk_x = wcx * WOOD_CELL_SIZE + wcx_off;
	result.trunk_z = wcz * WOOD_CELL_SIZE + wcz_off;

	result.trunk_x_offset = wcx_off;
	result.trunk_z_offset = wcz_off;

	int height_variation = 2 * sines[anglez % 180];
	// one block above the trunk + grass
	// trunk base (grass block position) needs to be recomputed in case of checking a cell that
	// does not correspond to the current chunk
	result.leaves_y_pos = 1 + height_variation + TREE_STANDARD_HEIGHT + evaluateGrass(result.trunk_x,
		result.trunk_z);

	return result;
}
```

From here, we can use some basic distance functions to make the leaves have different
patterns. For example, this one generates leaves in a sphere around the trunk.

```
// Divide the world into cells, each with exactly one tree, so that no two trees will be adjacent of each other
struct TreeCellInfo info;
int wcx = (int)(x / WOOD_CELL_SIZE) - tree_lut_x_offset; // wood cell x
int wcz = (int)(z / WOOD_CELL_SIZE) - tree_lut_z_offset; // wood cell z

// Retrieve info on the cell from LUT
info = treeLUT[wcx * TREE_LUT_SIZE + wcz];

// A tree is to be placed in this position if the coordinates are those of the tree of the current cell
bool wood = x == info.trunk_x && z == info.trunk_z && y > grassNoiseLUT[lut_index] && y <= info.leaves_y_pos;
bool leaf{false};

// Check placing of leaves

// Place leaves on top of trunk
if(wood) leaf = y > info.leaves_y_pos && y < info.leaves_y_pos+LEAVES_RADIUS;
else{
    // Place leaves around trunk
    if(!leaf) leaf = utils::withinDistance(x,y,z, info.trunk_x, info.leaves_y_pos, info.trunk_z, LEAVES_RADIUS);

    // Eventually search neighboring cells for trunks
    if(!leaf && wcx+1 < TREE_LUT_SIZE){
	info = treeLUT[(wcx+1) * TREE_LUT_SIZE + wcz];
	leaf = utils::withinDistance(x,y,z, info.trunk_x, info.leaves_y_pos, info.trunk_z, LEAVES_RADIUS);
    }

    if(!leaf && wcx-1 >= 0){
	info = treeLUT[(wcx-1) * TREE_LUT_SIZE + wcz];
	leaf = utils::withinDistance(x,y,z, info.trunk_x, info.leaves_y_pos, info.trunk_z, LEAVES_RADIUS);
    }

    if(!leaf && wcz-1 >= 0){
	info = treeLUT[wcx * TREE_LUT_SIZE + (wcz-1)];
	leaf = utils::withinDistance(x,y,z, info.trunk_x, info.leaves_y_pos, info.trunk_z, LEAVES_RADIUS);
    }

    if(!leaf && wcz+1 < TREE_LUT_SIZE){
	info = treeLUT[wcx * TREE_LUT_SIZE + (wcz+1)];
	leaf = utils::withinDistance(x,y,z, info.trunk_x, info.leaves_y_pos, info.trunk_z, LEAVES_RADIUS);
    }
}

if(wood) block = Block::WOOD;
if(leaf) block = Block::LEAVES;
```

The need to check neighbouring cells could be eliminated by ensuring trunks generate far enough from
the border of the cell to include the foliage. But I like it this way, I think it gives it a bit of
variation without the need for huge cells.


<img alt="Leaves!" src="/resources/projects/voxel2/trees-leaves.png"/> 
<img alt="Leaves!" src="/resources/projects/voxel2/trees-leavescloseup.png"/> 

We can even generate different patterns for the leaves, by changing the distance function.

<img alt="Different leaves!" src="/resources/projects/voxel2/trees-leavesvariation.png"/> 
