title: Notes about Voxel VRAM usage with Geometry Shader

<a href="/projects/voxel2/better-meshing.html">Linked to this article</a>


# Notes about Voxel VRAM usage with Geometry Shader

Standard world (seeds 12345). Camera centered at (512, 80, 512), RENDER_DISTANCE=16 CHUNK_SIZE=32

No geometry shader: 82MB

With geometry shader: 20MB

With geometry shader + byte for data: 8MB

On the Radeon HD 6850, using integer data causes a huge drop in performance
