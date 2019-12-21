---
title: Implementing a basic Arcball Camera.
categories:
- Camera
feature_image: "https://picsum.photos/id/875/2560/600"
---

### What's an Arcball Camera?

Arcball cameras/orbit cameras are a simple type of camera that rotates aroud a fixed center point. So for example when we move the cursor from left to right, bottom to top, we want the camera to be rotated on a sphere surrounding the object at the center of our scene.

###### Left/right Rotation example
![Left/right rotation](https://media.giphy.com/media/fsnKctjdBVQWo0pXLy/giphy.gif)  

###### Up/Down Rotation example
![Up/down rotation](https://media.giphy.com/media/YSZbmrRYLQhhFJ6kdP/giphy.gif)  

### How does it work ?
There are multiple way to implement this. For this case we are going to find the new position of the camera given a delta angle. The main idea is to rotate the camera using a pivot point. Here's the algortihm:
1 : Get the camera actual right vector, up vector, position, and pivot.
2 : Rotate the camera around the pivot point on the x axis.
3 : Using the updated camera position, rotate the camera aroung the pivot point on the y axis.

### Implementation

```cpp
glm::vec3 rightVector = app->camera.GetRightVector();
glm::vec3 zVector(0, 0, 1);
glm::vec4 position(app->camera.GetEye().x, app->camera.GetEye().y, app->camera.GetEye().z, 1);
glm::vec4 target(app->camera.GetLookAt().x, app->camera.GetLookAt().y, app->camera.GetLookAt().z, 1);

float dist = glm::distance2(zVector, glm::normalize(app->camera.GetEye() - app->camera.GetLookAt()));

float xAngle = (app->m_mouseDownPos.x - xPos) * (M_PI/300);
float yAngle = (app->m_mouseDownPos.y - yPos) * (M_PI/300);

if (dist < 0.01 && yAngle < 0 || 4.0 - dist < 0.01 && yAngle > 0)
    yAngle = 0;

glm::mat4x4 rotationMatrixY(1.0f);
rotationMatrixY = glm::rotate(rotationMatrixY, yAngle, rightVector);

glm::mat4x4 rotationMatrixX(1.0f);
rotationMatrixX = glm::rotate(rotationMatrixX, xAngle, zVector);

position = (rotationMatrixX * (position - target)) + target;

glm::vec3 finalPositionV3 = (rotationMatrixY * (position - target)) + target;

app->camera.SetCameraView(finalPositionV3, app->camera.GetLookAt(), zVector);
```

<!-- more -->
