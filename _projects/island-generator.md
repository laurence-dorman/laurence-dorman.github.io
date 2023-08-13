---
layout: post
title: Island Terrain Generator 🏝 🏭
description: DirectX application that generates island terrains using Perlin noise | C++ / hlsl
date: 2022-01-19
---

>**Disclaimer: This is university coursework.**

## Table of contents
{:.no_toc}
* TOC
{:toc}

&nbsp;

***

&nbsp;

## Video demonstration
<iframe width="560" height="315" src="https://www.youtube.com/embed/M0cWqjhEt0A" title="YouTube video player" frameborder="1" allowfullscreen></iframe>

&nbsp;

***

&nbsp;

## [Introduction](#introduction)

The application features iterative Perlin noise functions in combination with pixel shaders to create
believable and highly customizable island terrains.

## [Features](#features)

### Pixel shaders

The application features a terrain pixel shader which colours the terrain based on its height and slope. It blends between textures stored in registers and updates pixel normals based on texture normal maps.
In the dirt/grass levels, we check the gradient of the terrain using the dot product of the terrain normal and the world up vector. If we are below a certain threshold, we blend in the grass texture. This creates a nice transition between the dirt and grass textures on flatter areas of the terrain.

~~~hlsl
bool checkSlope(float3 norm, float ang, inout float diff)
{ // Calculate angle from normal
    float angSlope = degrees(acos(dot(norm, float3(0.f, 1.f, 0.f))));
    if (angSlope < ang) // if angle of slope is less than ang degrees, return true;
    {
        diff = ang - angSlope; // store difference in diff
        return true;
    }
    return false;
}
...
if (checkSlope(normal, 25.f, angDiff)) // if angle less than 25 deg, grass, else dirt
{
    float4 normColour = dirtNormal.Sample(sampler0, tex * 5.f);
    normal -= float3(normColour.x, normColour.y, normColour.z)
    angDiff = normalizeFloat(angDiff, 0.f, 25.f);
    return lerpVec(dirt, grass, angDiff);
}
~~~

### Procedural terrain generation

#### Faulting

The application features a faulting algorithm which attempts to imitate fault lines shifting the terrain up or down.

1. First, we will choose 2 points randomly on each of the edges of the terrain.
2. Then we make a line between the points.
3. We then iterate through each point in the terrain, drawing a new line from the position to the first random position on the edge.
4. Then, by taking the cross product of the first line and our new line, we can determine if we are on one side of the first line if the cross product is positive.

~~~cpp
int point1 = rand() % resolution;
int point2 = rand() % resolution;
const float scale = terrainSize / (float)resolution;
XMFLOAT3 position1 = XMFLOAT3(0.f, heightMap[point1] * scale, point1 * scale);
XMFLOAT3 position2 = XMFLOAT3(terrainSize * scale, heightMap[point2] * scale, point2 * scale);
XMFLOAT3 line = XMFLOAT3(position2.x - position1.x, position2.y - position1.y, position2.z - position1.z);
float add = (rand() % 2 == 0) ? 1.f : -1.f; // decide whether we will raise or lower the terrain randomly (50/50)
for (int j = 0; j < (resolution); j++) {
    for (int i = 0; i < (resolution); i++) {
        XMFLOAT3 currentPos = XMFLOAT3(i * scale, heightMap[(j * resolution) + i] * scale, j * scale);
        XMFLOAT3 newLine = XMFLOAT3(currentPos.x - position1.x, currentPos.y - position1.y, currentPos.z - position1.z);
        float y_cross = ((line.z * newLine.x) - (line.x * newLine.z));
        if (y_cross > 0.f)
            heightMap[(j * resolution) + i] += add;
    }
}
~~~

#### Smoothing

The application features a smoothing algorithm, which smoots the terrain by calculating the average height for each point in the height map using it's "Moore neighbourhood" (the 8 points surrounding it).

~~~cpp
int current = (j * resolution) + i;
float points[9] = { current - 1 - resolution, current - resolution, current - resolution + 1,
                    current - 1, current, current + 1,
                    current - 1 + resolution, current + resolution, current + resolution + 1,
                  };
~~~

#### Perlin noise

The application features a Perlin noise generation function with methods adapted from [Ken Perlin's Improved Noise](https://mrl.cs.nyu.edu/~perlin/noise/). It generates a permutation table on initialisation which is used to get pseudo-random numbers inside the noise functions. It uses the pseudo-random numbers to generate pseudo-random vectors, from our position inside the 3D cube, to the 8 corners of the cube, which have a psuedo-random value from the permutation table. We also calculate the vectors from our unit position inside the cube to the 8 corners. We then interpolate between these 8 gradients using Ken Perlin's easing cube defined by the following function:

$$6t^5 - 15t^4 + 10t^3$$

Using this ease curve makes interpolation between the gradients smoother, and therefore the noise more natural.

Adding octaves to the noise function allows us to add more detail to the noise. We can do this by adding the noise function to itself, but with a different frequency and amplitude. We can adjust the effect the Perlin noise function has on the terrain with parameters like scale, amplitude and octaves.

~~~cpp
float height = 0.f;
point.x /= scale; point.y /= scale;
for (int i = 0; i < octaves; i++) {
    height += (amplitude * noise(point, translation)); // increment height by perlin noise
    point.x *= 2.f; point.y *= 2.f; // move positions
    amplitude *= 0.5f; // half amplitude (higher iterations have less impact and arent increasing overall height)
}
return height;
~~~

#### UI

The application features a basic UI which allows the user to adjust the parameters of the terrain generation, change the mode and change lighting/rendering. The UI is created using [ImGui](https://github.com/ocornut/imgui).

[![UI](/assets/images/island-generator/ui.png)](/assets/images/island-generator/ui.png)

&nbsp;

***

&nbsp;

## [Code architecture](#code-architecture)

The perlin noise functions are contained within the `Perlin` class, which means we can call the functions from anywhere in the application to get the "height" or noise value of any position, supplying the function with a position and translation. The iterative Perlin noise function takes parameters: `octaves`, `amplitude` and `scale`. THe function is static, so we can call it without instantiating the `Perlin` class.

~~~cpp
float height = Perlin::fbm(XMFLOAT3(positionX, positionZ, 0.f), translation, octaves, amplitude, scale);
~~~

The `BuildHeightMap` function contains a switch statement based ont he type of terrain modification we want to do e.g. faulting, smoothing, perlin noise

~~~cpp
switch (type) {
	case BuildType::FLAT:
		Flat();
		break;
	case BuildType::SMOOTH:
	{
		Smooth();
		break;
	}
	case BuildType::FAULT:
	{
		Fault();
		break;
	}
	case BuildType::FBM:
		Fbm(perlinTranslation, perlinOctaves, perlinAmplitude, perlinScale, perlinFalloff);
		break;
	}
~~~

This allows us to simply call `setBuildType` from the main application class and then `Regenerate`. This means we can quickly change generation types using the UI:

~~~cpp
if (ImGui::Button("FBM Mode")) {
		m_Terrain->setBuildType(m_Terrain->FBM);
		m_Terrain->Regenerate(renderer->getDevice(), renderer->getDeviceContext());
	}
~~~

&nbsp;

***

&nbsp;

## [Sources](#sources)

- [Improved Noise – Ken Perlin](https://mrl.cs.nyu.edu/~perlin/noise) (Accessed 18/01/2022).
- [Perlin Noise: A Procedural Generation Algorithm](https://rtouti.github.io/graphics/perlin-noise-algorithm) (Accessed 18/01/2022) - Improved my understanding of Ken Perlin’s Noise implementation.
- [Fractal Brownian Motion](https://thebookofshaders.com/13) (Accessed 18/01/2022). Improved my understanding on how to implement iterative Perlin noise generation using octaves.

&nbsp;

***

&nbsp;

## [Resource references](#resource-references)

- Water textures [“Water_001”](https://3dtextures.me/2017/12/28/water-001/) (Accessed 13/01/2022).
- Rock textures [“Dirt 006”](https://3dtextures.me/2020/04/27/dirt-006/) (Accessed 18/01/2022).
- Grass textures [“Stylized grass 001”](https://3dtextures.me/2020/06/02/stylized-grass-001/) (Accessed 18/01/2022).
- Dirt textures [“Forest ground 002”](https://3dtextures.me/2019/12/10/forest-ground-002/) (Accessed 18/01/2022).
- Sand textures [“Sand 005”](https://3dtextures.me/2020/02/14/sand-005/) (Accessed 18/01/2022).
- Snow textures [“Snow 003”](https://3dtextures.me/2018/03/01/snow-003/) (Accessed 18/01/2022).
