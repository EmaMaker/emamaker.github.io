title: Voxel Engine (again) - code snippets

# Generation {#generation}

```
std::array<int, CHUNK_SIZE * CHUNK_SIZE> grassNoiseLUT;
std::array<int, CHUNK_SIZE * CHUNK_SIZE> dirtNoiseLUT;

void generateNoise(Chunk::Chunk *chunk)
{
    for (int i = 0; i < grassNoiseLUT.size(); i++)
    {
        grassNoiseLUT[i] = -1;
        dirtNoiseLUT[i] = -1;
    }

    Block block_prev{Block::AIR};
    int block_prev_start{0};

    // A space filling curve is continuous, so there is no particular order
    for (int s = 0; s < CHUNK_VOLUME; s++)
    {

        int x = HILBERT_XYZ_DECODE[s][0] + CHUNK_SIZE * chunk->getPosition().x;
        int y = HILBERT_XYZ_DECODE[s][1] + CHUNK_SIZE * chunk->getPosition().y;
        int z = HILBERT_XYZ_DECODE[s][2] + CHUNK_SIZE * chunk->getPosition().z;
        int d2 = HILBERT_XYZ_DECODE[s][0] * CHUNK_SIZE + HILBERT_XYZ_DECODE[s][2];

        if (grassNoiseLUT[d2] == -1){
            grassNoiseLUT[d2] = GRASS_OFFSET + (int)((0.5 + noiseGen1.eval(x * NOISE_GRASS_X_MULT, z * NOISE_GRASS_Z_MULT) * NOISE_GRASS_MULT));
	}
        if (dirtNoiseLUT[d2] == -1){
            dirtNoiseLUT[d2] = NOISE_DIRT_MIN + (int)((0.5 + noiseGen2.eval(x * NOISE_DIRT_X_MULT, z * NOISE_DIRT_Z_MULT) * NOISE_DIRT_MULT));
	}

        int grassNoise = grassNoiseLUT[d2];
        int dirtNoise = dirtNoiseLUT[d2];
        int stoneLevel = grassNoise - dirtNoise;

        if (y < stoneLevel)
            block = Block::STONE;
        else if (y >= stoneLevel && y < grassNoise)
            block = Block::DIRT;
        else if (y == grassNoise)
            block = Block::GRASS;
        else
            block = Block::AIR;

        if (block != block_prev)
        {
            chunk->setBlocks(block_prev_start, s, block_prev);
            block_prev_start = s;
        }

        block_prev = block;
    }

    chunk->setBlocks(block_prev_start, CHUNK_VOLUME, block_prev);
    chunk->setState(Chunk::CHUNK_STATE_GENERATED, true);
}
```

# Concentric circles
```
	int index{0};
	int rr{RENDER_DISTANCE * RENDER_DISTANCE};

        int xp{0}, x{0};
        bool b = true;

        // Iterate over all chunks, in concentric spheres starting fron the player and going outwards. Alternate left and right
        // Eq. of the sphere (x - a)² + (y - b)² + (z - c)² = r²
        while (xp <= RENDER_DISTANCE)
        {
	    // Alternate between left and right
            if (b) x = +xp;
            else x = -xp;

	    // Step 1. At current x, get the corresponding y values (2nd degree equation, up to 2
	    // possible results)
            int y1 = static_cast<int>(sqrt((rr) - x*x));

            for (int y = -y1 + 1 ; y <= y1; y++)
            {
                // Step 2. At both y's, get the corresponding z values
		int z1 = static_cast<int>(sqrt( rr - x*x - y*y));

                for (int z = -z1 + 1; z <= z1; z++){
		    chunks_indices[index][0] = x;
		    chunks_indices[index][1] = y;
		    chunks_indices[index][2] = z;
		    index++;
		}
            }

            if (!b)
            {
                xp++;
                b = true;
            }
            else  b = false;
	}
	chunks_volume_real = index;

```
