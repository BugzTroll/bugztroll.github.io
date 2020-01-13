---
title: How to make an infinite grid.
description: Adding an infinite grid to a scene using shaders
categories:
- Scene Helper
feature_image: "https://i.imgur.com/ql5751o.png"
# feature_text: "A Slice Of Rendering"
aside: true
author: marie
---

*Keywords: Vulkan, C++, Shader programming, glsl, Infinite grid, Scene helper, Axis, Coordinate system*.

After struggling with my coordinate system when importing my models in my scene, I decided to add an infinite grid with the main axis highlighted. It's always usefull to have visualisation debugging tools anyway so I thought this was an important feature to help with the next steps.

At first I thought it would be easy peasy lemon squeezy but... it was a little more tricky then I expected. Here's the details.

![Infinite grid](https://media.giphy.com/media/ka6g95y9r8HzdpcUA0/giphy.gif)

### Algorithm
1. Create a basic plane vertex and fragment shader with the coordinates of the grid
2. Unproject the plane coordinates so that the points are always at infinity
3. Draw the plane only when it intersects the floor (ie $$y = 0$$)
4. Draw the grid and the axis highlight (red for x and blue for z)
5. Output the grid depth for every fragment
5. Add a fade out effect when the grid is far away

### Step 1 : Plane Basic shaders

The first thing we need to do is to draw a simple xy plane like illustrated below.

![Grid step 1](https://i.imgur.com/Jy3GMka.png)

##### Vertex shader with plane position

Here's my initial vertex shader to draw the plane.
```glsl
// Shared set between most vertex shaders
layout(set = 0, binding = 0) uniform ViewUniforms {
    mat4 view;
    mat4 proj;
    vec3 pos;
} view;

// Grid position are in xy clipped space
vec3 gridPlane[6] = vec3[](
    vec3(1, 1, 0), vec3(-1, -1, 0), vec3(-1, 1, 0),
    vec3(-1, -1, 0), vec3(1, 1, 0), vec3(1, -1, 0)
);
// normal vertice projection
void main() {
    gl_Position = view.proj * view.view * vec4(gridPlane[gl_VertexIndex].xyz, 1.0);
}

```
##### Fragment shader

And my fragment shader, simply outputing the color red for now.

```glsl
layout(location = 0) out vec4 outColor;

void main() {
    outColor = vec4(1.0, 0.0, 0.0, 1.0);
}
```

### Step 2 : Unproject clipped space points

The next thing we need to do is to use directly the grid coordinate like if it was in clipped space coordinate and unproject it to get the 3D world space coordinates.
Since we want our points to be at infinity, we need to unproject it on the the near (z = 0) and far (z = 1) planes.

We are also going to use directly the clipped space grid coordinate instead of applying a viewProjection to the coordinate in the vertex shader.
The near and far points are going to be used in the fragment shader for the next steps, so we are going to pass them.

After doing this we will have an infinite red plane covering the entire viewport like illustrated below.

![Grid step 2](https://i.imgur.com/wK0jckw.png)

##### Vertex shader update

Here's the updated shader with the functions to unproject the points.

```glsl
layout(set = 0, binding = 0) uniform ViewUniforms {
    mat4 view;
    mat4 proj;
    vec3 pos;
} view;

layout(location = 1) out vec3 nearPoint;
layout(location = 2) out vec3 farPoint;

// Grid position are in clipped space
vec3 gridPlane[6] = vec3[] (
    vec3(1, 1, 0), vec3(-1, -1, 0), vec3(-1, 1, 0),
    vec3(-1, -1, 0), vec3(1, 1, 0), vec3(1, -1, 0)
);

vec3 UnprojectPoint(float x, float y, float z, mat4 view, mat4 projection) {
    mat4 viewInv = inverse(view);
    mat4 projInv = inverse(projection);
    vec4 unprojectedPoint =  viewInv * projInv * vec4(x, y, z, 1.0);
    return unprojectedPoint.xyz / unprojectedPoint.w;
}

void main() {
    vec3 p = gridPlane[gl_VertexIndex].xyz;
    nearPoint = UnprojectPoint(p.x, p.y, 0.0, view.view, view.proj).xyz; // unprojecting on the near plane
    farPoint = UnprojectPoint(p.x, p.y, 1.0, view.view, view.proj).xyz; // unprojecting on the far plane
    gl_Position = vec4(p, 1.0); // using directly the clipped coordinates
}
```

### Step 3 : Draw the plane when it intersects the floor

Since the plane is now drawn on the entire viewport, we need to make some calculation to make sure it's only visible when needed, meaning we only want to see the plane on the floor (when $$y = 0$$). 

We are going to use the far and near point calculated earlier to check if the plane intersects with the floor. Given the parametric equation of a line :

$$
\begin{align*}
 y = nearPoint.y + t \times (farPoint.y - nearPoint.y)
\end{align*}
$$

if we isolate t and set $$y = 0$$ we now have :

$$
\begin{align*}
 t = \frac{-nearPoint.y} {farPoint.y - nearPoint.y}
\end{align*}
$$

and the plane should only be visible when $$t > 0$$. After adding this condition, we should see a plane like this :

![Grid step 3](https://i.imgur.com/bwHq3ph.png)

##### Update in fragment shader

Here is the updated fragment shader using the t value to set the visibility of the plane

```glsl
layout(location = 1) in vec3 nearPoint; // nearPoint calculated in vertex shader
layout(location = 2) in vec3 farPoint; // farPoint calculated in vertex shader
layout(location = 0) out vec4 outColor;
void main() {
    float t = -nearPoint.y / (farPoint.y - nearPoint.y);
    outColor = vec4(1.0, 0.0, 0.0, 1.0 * float(t > 0)); // opacity = 1 when t > 0, opacity = 0 otherwise
}
```
### Step 4 : Draw the actual grid

We now have an infinite plane, but what we really want is an infinite grid. We need to make some calculation to draw lines instead of just a uniform color. We also need to add some antialiasing to make it look good. 

We are going to compute the 3D position on the actual xz plane using the nearPoint and farPoint calculated earlier and use that position to determine if the point is actually on a line or on the void of the grid. For the AA, we are going to use screen-space partial derivatives to compute the line width and falloff.

Here's the result we are supposed to have after doing that.

![Grid step 4](https://i.imgur.com/1RVpEUf.png)

##### Fragment shader update

Here's the fragment shader updated with the calculation of the 3D point.

```glsl
layout(location = 1) in vec3 nearPoint;
layout(location = 2) in vec3 farPoint;
layout(location = 0) out vec4 outColor;

vec4 grid(vec3 fragPos3D, float scale) {
    vec2 coord = R.xz * scale; // use the scale variable to set the distance between the lines
    vec2 derivative = fwidth(coord);
    vec2 grid = abs(fract(coord - 0.5) - 0.5) / derivative;
    float line = min(grid.x, grid.y);
    float minimumz = min(derivative.y, 1);
    float minimumx = min(derivative.x, 1);
    vec4 color = vec4(0.2, 0.2, 0.2, 1.0 - min(line, 1.0));
    // z axis
    if(R.x > -0.1 * minimumx && R.x < 0.1 * minimumx)
        color.z = 1.0;
    // x axis
    if(R.z > -0.1 * minimumz && R.z < 0.1 * minimumz)
        color.x = 1.0;
    return color;
}
void main() {
    float t = -nearPoint.y / (farPoint.y - nearPoint.y);
    vec3 fragPos3D = nearPoint + t * (farPoint - nearPoint);
    outColor = grid(fragPos3D, 10) * float(t > 0);
}
```

### Step 5 : Output depth

Since the plane is drawn on the entire viewport, we now have some problems when drawing other objets in the scene. What we need to do to fix that is to manually calculate and output the depth for every fragment. 

To do this we are going to need the view and projection matrix, so don't forget to pass them from the vertex shader to the fragment shader.

![Grid step 5](https://i.imgur.com/t1k8TYY.png)

##### Fragment shader update

```glsl
layout(location = 1) in vec3 nearPoint;
layout(location = 2) in vec3 farPoint;
layout(location = 3) in mat4 fragView;
layout(location = 7) in mat4 fragProj;
layout(location = 0) out vec4 outColor;

vec4 grid(vec3 fragPos3D, float scale, bool drawAxis) {
    vec2 coord = R.xz * scale;
    vec2 derivative = fwidth(coord);
    vec2 grid = abs(fract(coord - 0.5) - 0.5) / derivative;
    float line = min(grid.x, grid.y);
    float minimumz = min(derivative.y, 1);
    float minimumx = min(derivative.x, 1);
    vec4 color = vec4(0.2, 0.2, 0.2, 1.0 - min(line, 1.0));
    // z axis
    if(R.x > -0.1 * minimumx && R.x < 0.1 * minimumx)
        color.z = 1.0;
    // x axis
    if(R.z > -0.1 * minimumz && R.z < 0.1 * minimumz)
        color.x = 1.0;
    return color;
}
float computeDepth(vec3 pos) {
    vec4 clip_space_pos = fragProj * fragView * vec4(pos.xyz, 1.0);
    return (clip_space_pos.z / clip_space_pos.w);
}
void main() {
    float t = -nearPoint.y / (farPoint.y - nearPoint.y);
    vec3 fragPos3D = nearPoint + t * (farPoint - nearPoint);
    gl_FragDepth = computeDepth(fragPos3D);
    outColor = grid(fragPos3D, 10, true) * float(t > 0);
}
```

### Step 6 : Fade out the grid

Almost done ! Now we just need to add a fading effect so that the grid looks a little better when it's far away. To do this, we actually need to use the linear depth to determine the alpha of the lines (The more far away they are, the more transparent they will be).
To get the linear depth we will need our real near and far plane values (don't forget to bind them to the vertex shader and pass them to the fragment shader)

![Grid step 6](https://i.imgur.com/AcgzIZD.png)

##### Fragment shader update

Here's the last version of the fragment shader ! Finally :)

```glsl
layout(location = 0) in float near; //0.01
layout(location = 1) in float far; //100
layout(location = 2) in vec3 nearPoint;
layout(location = 3) in vec3 farPoint;
layout(location = 4) in mat4 fragView;
layout(location = 8) in mat4 fragProj;
layout(location = 0) out vec4 outColor;

vec4 grid(vec3 fragPos3D, float scale, bool drawAxis) {
    vec2 coord = R.xz * scale;
    vec2 derivative = fwidth(coord);
    vec2 grid = abs(fract(coord - 0.5) - 0.5) / derivative;
    float line = min(grid.x, grid.y);
    float minimumz = min(derivative.y, 1);
    float minimumx = min(derivative.x, 1);
    vec4 color = vec4(0.2, 0.2, 0.2, 1.0 - min(line, 1.0));
    // z axis
    if(R.x > -0.1 * minimumx && R.x < 0.1 * minimumx)
        color.z = 1.0;
    // x axis
    if(R.z > -0.1 * minimumz && R.z < 0.1 * minimumz)
        color.x = 1.0;
    return color;
}
float computeDepth(vec3 pos) {
    vec4 clip_space_pos = fragProj * fragView * vec4(pos.xyz, 1.0);
    return (clip_space_pos.z / clip_space_pos.w);
}
float computeLinearDepth(vec3 pos) {
    vec4 clip_space_pos = fragProj * fragView * vec4(pos.xyz, 1.0);
    float clip_space_depth = (clip_space_pos.z / clip_space_pos.w) * 2.0 - 1.0; // put back between -1 and 1
    float linearDepth = (2.0 * near * far) / (far + near - clip_space_depth * (far - near)); // get linear value between 0.01 and 100
    return linearDepth / far; // normalize
}
void main() {
    float t = -nearPoint.y / (farPoint.y - nearPoint.y);
    vec3 fragPos3D = nearPoint + t * (farPoint - nearPoint);

    gl_FragDepth = computeDepth(fragPos3D);

    float linearDepth = computeLinearDepth(fragPos3D);
    float fading = max(0, (0.5 - linearDepth));

    outColor = (grid(fragPos3D, 10, true) + grid(fragPos3D, 1, true))* float(t > 0); // adding multiple resolution for the grid
    outColor.a *= fading;
}
```
<!-- more -->
