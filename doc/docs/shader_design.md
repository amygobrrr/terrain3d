Shader Design
==============

Our shader combines a lot of ideas and code from [cdxntchou's IndexMapTerrain](https://github.com/cdxntchou/IndexMapTerrain) for Unity, [Zylann's HTerrain](https://github.com/Zylann/godot_heightmap_plugin/) for Godot, the Witcher 3 talk linked in the System Design page, and our own thoughts and optimizations.

## Uniforms

[Terrain3DMaterial](../api/class_terrain3dmaterial.rst) exposes uniforms found in the shader, whether we put them there or you do with your own custom shader. Uniforms that begin with `_` are considered private and are not exposed. However you can access them via code. You can create your own private uniforms.

These notable [Terrain3DStorage](](../api/class_terrain3dstorage.rst) variables are passed in as uniforms:
* `_region_map`, `_region_offsets` define the location and IDs of regions (sculpted areas)
* `_height_maps`, `_control_maps`, and `_color_maps` texture arrays define the elevation, textures, and colors of the terrain
* `_texture_array_albedo`, `_normal` are the texture arrays compiling all individual textures

## Vertex() & Supporting functions

`vertex()` is run per terrain mesh vertex drawn on the screen.

First are `get_region_uv/_uv2()` which take in UV coordinates and return region coordinates, either absolute or normalized. It also returns the region ID, which is used in the map texture arrays above.

Optionally, world noise is inserted here, which generates fractal brownian noise to be used for background hills outside of your regions. It's a visual gimmick only and does not generate collision.

`get_height()` returns the value of the heightmap at the given location. If world noise is enabled, it is blended into the height here.

Finally `vertex()` sets the UV and UV2 coordinates, and the height of the mesh vertex. Elsewhere the CPU creates flat meshe components and a mountainous collision mesh. Here is where the flat meshe vertices have their heights set to match the collision mesh with `VERTEX.y = get_height(UV2)`.

## Fragment()

`fragment()` is run per terrain pixel drawn on the screen.

### Normal calculation

The first step is calculating the terrain normals. This is not done at all in `vertex()`. On other terrain systems there might be optimizations to be had by doing so, however this does not work on a clipmap terrain because the vertices spread out on lower LODs. Certain things like normals look strange when you look in the distance and the vertices used for calculation suddenly separate on further LODs. So we calculate normals per pixel.

Normally, generating normals in the shader works fine and modern GPUs can handle the load of 4 additional height lookups and the on-the-fly calculations. However, because we do some processing by region, there is an evasive bug in [#185](https://github.com/TokisanGames/Terrain3D/issues/185) that is caused by the GPU's linear interpolation of the edges of our region maps when calculating heights using pixels across region boundaries. So we may have to switch to generating normal maps on the CPU and passing them in as a texture as many terrain systems do. The difference is 4 fewer texture lookups per pixel vs consuming another 3-4MB VRAM per region.

### Grid creation

We create a grid 1 unit wide using the `mirror` and `index00UV`-`index11UV` variables. This defines 4 fixed points around the current pixel. On LOD0 this grid aligns with both the mesh vertices and the control map pixels. However, they don't align further out on lower LODs or beyond the regions (sculpted areas) where the vertices are spread out. Pixel processing out there still occurs based on this 1-unit grid.

This is `ALBEDO = vec3(mirror.xy, 0.)`, showing horizontal stripes in red, and vertical stripes in green, with the vertices highlighted. The inverse is stored in `mirror.zw`, so where horizontal stripes alternate red, black, red, they are now black, red, black.

```{image} images/sh_mirror.png
:target: ../_images/sh_mirror.png
```

Next, the control maps are queried for each of the 4 grid points and stored in `control00`-`control11`. The control map bit packed format is defined in [Controlmap Format](controlmap_format.md). 

The textures at each point are looked up and stored in an array of structs. If there is an overlay texture, the two are height blended here. Then the pixel position within the 4 grid points is bilinear interpolated to get the weighting for the final pixel values.

_Side note_: Interestingly, since the grid aligns with vertices on LOD0, there is potential for optimizations with control/texture lookups and normal calculations. For LOD0 only, all three could be looked up in `vertex()` instead of per pixel. Sadly, this breaks down on lower LODs due to the vertices spreading apart. It would be interesting to experiment with a very large LOD0 and see if the savings on lookups per pixel outweighs the more complex mesh, since the pixel shader is a lot slower than drawing vertices. Perhaps that efficiency would also be worth replacing the entire clipmap nature of the terrain system with another method to overcome other issues.

### Texture Sampling - Splat map vs Index map

It is trivial to store one texture value for any given location on a control map. However, what if we want to blend two or more textures at that point? How do we identify up to 16 or 32 textures? How do we blend them?

This analysis compares the *splat map* method used by [Zylann's HTerrain](https://github.com/Zylann/godot_heightmap_plugin/) and other systems with an *index map* method, used by [cdxntchou's IndexMapTerrain](https://github.com/cdxntchou/IndexMapTerrain).

At their core, all height map based terrain tools are just fancy painting applications. For texturing, the reasonable approach is to paint each texture on the control map and blend with opacity exactly as one would in Photoshop or Krita. This is what HTerrain does, but instead of painting with just RGBA, it paints with RGBACDEFHIJKLMNO. The unreasonable approach would be to use an entirely different methodology in order to reduce memory or increase speed.

**HTerrain's splat map approach** specifies 16 textures at each terrain vertex. This is stored in 4 splat maps, each consuming 32-bits as RGBA. This means each vertex in the terrain uses 4 bytes per 4 splat maps, for a total of 16 bytes for texture values. Each value represents a texture strength of 0-255. 

All 4 splat maps are sampled, and of the 16 values, the 4 strongest are height blended together, per terrain pixel. The blending of textures for pixels drawn between vertices is handled by the GPU's linear interpolated texture filter during texture lookups.

**Our index map approach** samples a control map at 4 fixed grid points surrounding the current terrain pixel. The 4 surrounding samples have a base texture, an overlay texture, and a blending value. The base and overlay texture values range from 0-31, each stored in 5 bits. The blend values are stored in 8-bits.

*Side note:* Storing blend values in 3-bits is possible, where each of the 8 numbers represents an index in a array of 0-1 values: `{ 0.0f, .125f, .25f, .334f, .5f, .667f, .8f, 1.0f }`. In the future, this may be baked at runtime. However, editing using a 3-bit array of fixed values was exceedingly difficult and unsuccessful.

The position of the pixel within its grid square is used to bilinear interpolate the values of the 4 surrounding samples. We disable the default GPU interpolation on texture lookups and interpolate ourselves here. 

**Comparing the two methods:**

* **Texture lookups** - Considering only lookups for which ground texture to use and loading the texture data:
  * HTerrain uses 12 lookups per pixel:
    * 4 for the 4 splat maps to get the texture for the closest vertex point
    * 8 for the strongest 4 albedo_height textures and the 4 normal_rough textures
  * Terrain3D uses 12-20 lookups per pixel:
    * 4 for the surrounding 4 grid points on the control map
    * 8-16 for the 2-4 albedo_height & normal_rough for the base and overlay textures, for each of the 4 grid points

* **VRAM consumed** - HTerrain stores 16 texture strength values in 16 bytes per pixel (16 * 8 bits = 128 bits). We could store that in 16 bits. However, we provide 32 textures. Using the splat map method, 32 textures would require 32 bytes per pixel. We store it in 18 bits (5 base, 5 overlay, 8 blend value). On a 4096 x 4096 terrain with 16M pixels, HTerrain consumes 256MB for a 16 texture splatmap; 32 texture splatmap would require 512MB. We can designate 32 textures in 36MB. This calculation considers only the portion of the control maps that define which textures to use. Both tools use up a lot more VRAM for other things.

**In practice**

* HTerrain - 4 textures can technically be blended per vertex. Generally, painting 3-4 textures in an area works fine. However, some combinations of 3 textures (depending on their position in the texture array) and sometimes adding in a 4th introduces artifacts. 

* Terrain3D - Getting 3 or 4 textures in an area is feasible as long as only 2 textures are blending per grid point (vertex). It's possible to achieve a natural looking result with [the right technique](texture_painting.md#manual-painting-technique).

### Calculating weights

Since this pixel exists within four points on a grid, we can use bilinear interpolation to calculate weights based on how close we are to the grid points. e.g. The current pixel is 75% to the next X and 33% to the next Y. We combine these values with the blended texture height value to calculate our final weights.

### Applying PBR

Lastly, we calculate our final PBR values by using a weighted average of the four surrounding grid points.

The color map is looked up. Then all PBR values are sent to the GPU.

